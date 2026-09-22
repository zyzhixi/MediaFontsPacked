# MediaFontsPacked（.mfp）

> **多媒体字体包格式规范 · v1.0.0**
> 把多个 TTF / OTF / TTC / OTC 字体打包进单文件容器，支持内容寻址、增量更新与按需安装。

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)
[![Format Version](https://img.shields.io/badge/formatVersion-1-blue.svg)](./SPEC.md)
[![Spec](https://img.shields.io/badge/规范-v1.0.0-orange.svg)](./SPEC.md)

---

## 这是什么

`.mfp`（**M**edia **F**onts **P**acked）是一个开放的**容器格式规范**，用于把一批字体及其元数据、
预览图、许可证文本打包成单个文件，并支持在不重写整个文件的前提下做增量更新。

本仓库是**协议层规范**，不是某个客户端的实现。它定义字节布局、字段语义、兼容规则与校验流程，
把增量更新算法、恢复记录生成、预览渲染、字体安装等留给客户端自行决定
（完整职责边界见 [`SPEC.md`](./SPEC.md) 第 2.2 节）。

### 设计上最重要的一点

`.mfp` 基于**标准 ZIP64** 构建。**7-Zip、WinRAR、`unzip` 可以直接打开并解压它。**

这是刻意的设计选择，不是缺陷：

| 换来什么   | 说明                                     |
| ------ | -------------------------------------- |
| 生态互操作性 | 用户用任何熟悉的工具都能查看、备份、迁移包内资源，不被单一客户端绑定    |
| 可排障性   | 出问题时能直接用通用工具定位是容器结构问题还是客户端逻辑问题        |
| 零成本迁移  | 已有 ZIP 工具链的团队可直接复用现有流程生成与校验容器         |
| 快速识别   | 文件头有固定 magic，`manifestOffset` 提供直达索引的路径 |

内容完整性由**逐条目 SHA-256**（`checksum` 结构）与可选的**签名条目**（`.mfp-signature`）承担。

> 如果你的需求是「物理上无法被解压」，本格式不适用——见 [`SPEC.md`](./SPEC.md) 1.5 节。

---

## 核心能力

| 能力        | 说明                                              |
| --------- | ----------------------------------------------- |
| 单文件承载多字体  | TTF / OTF / TTC / OTC / WOFF / WOFF2，按需安装单个或全部  |
| 内容寻址与去重   | 字体条目以内容 SHA-256 命名，天然去重且可校验完整性                  |
| 支持超大文件    | 强制 ZIP64，单容器可超过 4 GB                            |
| 流式读取与随机访问 | 中央目录在文件末尾，读索引即可按需提取单个条目                         |
| 高频增量更新    | 通过 `revision` 链与 ZIP 追加语义，新增/删除字体只重写中央目录        |
| 跨平台可移植    | 格式本身不含平台相关内容，安装策略由客户端决定                         |
| 向后兼容      | 新增可选字段不影响旧客户端读取                                 |
| 分级降级      | 可选字段缺失时按既定规则降级，不拒绝读取                            |

---

## 30 秒速览

**文件结构**（最小容器，无恢复记录）：

```
偏移 0
┌────────────────────────────────────────────────┐
│  MFP Header（固定 32 字节，magic = "MFP\x01"）     │
├────────────────────────────────────────────────┤
│  Local File Header + manifest.json（deflate）   │  ← manifestOffset 指向这里
├────────────────────────────────────────────────┤
│  Local File Header + fonts/<sha256>.ttf（store） │
├────────────────────────────────────────────────┤
│  Central Directory → ZIP64 EOCD → Locator → EOCD │
└────────────────────────────────────────────────┘ 文件末尾
```

**manifest 最小示例**（完整版见 [`examples/full.json`](./examples/full.json)）：

```json
{
  "mfpVersion": "1.0.0",
  "formatVersion": 1,
  "revision": 1,
  "parentRevision": null,
  "createdAt": "2026-09-20T12:00:00Z",
  "updatedAt": "2026-09-20T12:00:00Z",
  "operation": "create",
  "package": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "My Font Pack",
    "version": "1.0.0"
  },
  "fonts": [
    {
      "id": "font-001",
      "hash": "sha256:2b1a7c9e...2f1a0b",
      "file": "fonts/2b1a7c9e...2f1a0b.ttf",
      "originalName": "MyFont-Regular.ttf",
      "format": "ttf",
      "isCollection": false,
      "faceCount": 1,
      "installMode": "singleFace",
      "metadata": {
        "familyName": "My Font",
        "subfamilyName": "Regular",
        "postscriptName": "MyFont-Regular",
        "fullName": "My Font Regular"
      }
    }
  ],
  "checksum": {
    "algorithm": "sha256",
    "manifestHash": "sha256:...",
    "entries": {
      "fonts/2b1a7c9e...2f1a0b.ttf": "sha256:2b1a7c9e...2f1a0b"
    }
  }
}
```

**实现者最容易踩的坑**：把 32 字节 MFP Header 前置到 ZIP 数据之后，**必须**同步修正 ZIP 结构中
5 处偏移字段（各 `+= 32`）。漏改会让通用解压工具失败，而 MFP 客户端自己可能仍"看起来正常"——
症状极其隐蔽。详见 [`SPEC.md`](./SPEC.md) 3.3.4 与 [`examples/layout.md`](./examples/layout.md)。

---

## 仓库结构

```
MediaFontsPacked/
├── README.md                    # 本文件
├── SPEC.md                      # 格式规范（协议层，权威定义）
├── IMPLEMENTATION.md            # 客户端实现指南（模块划分、接口签名、伪代码）
├── CHANGELOG.md                 # 各版本变更记录
├── CONTRIBUTING.md              # 贡献指南
├── LICENSE                      # MIT
├── schema/
│   ├── manifest.schema.json     # manifest 的 JSON Schema（draft 2020-12）
│   └── template.schema.json     # 预览模板的 JSON Schema
└── examples/
    ├── minimal.json             # 最小可用 manifest（单字体）
    ├── full.json                # 完整 manifest（集合字体 + 模板 + 冲突 + 恢复记录）
    ├── template.default-v1.json # 默认预览模板
    └── layout.md                # 最小文件结构的字节级示意
```

---

## 文档导航

| 我想…                | 去看                                                            |
| ------------------ | ------------------------------------------------------------- |
| 知道文件长什么样、每个字段什么含义 | [`SPEC.md`](./SPEC.md) 第 3 章（物理布局）、第 4 章（manifest）            |
| 写一个读写 `.mfp` 的客户端  | [`IMPLEMENTATION.md`](./IMPLEMENTATION.md) 第 3、4、6 章           |
| 校验一个 manifest 是否合法 | [`schema/manifest.schema.json`](./schema/manifest.schema.json) |
| 看一个能跑通的最小例子       | [`examples/minimal.json`](./examples/minimal.json)            |
| 理解增量更新怎么工作        | [`SPEC.md`](./SPEC.md) 4.2（revision 链）+ 实现指南 6.3               |
| 知道该报什么错误码         | [`SPEC.md`](./SPEC.md) 10.3（错误码表）                              |
| 确认旧版本文件怎么兼容       | [`SPEC.md`](./SPEC.md) 第 6 章（兼容规则、降级表、兼容性矩阵）                  |
| 排查 7-Zip 解压失败      | [`SPEC.md`](./SPEC.md) 3.3.4 + [`examples/layout.md`](./examples/layout.md) |

---

## 给实现者的三条建议

1. **先跑通 P0（只读 + 安装）**，再上打包，最后做增量更新与恢复记录。
   分阶段清单见 [`IMPLEMENTATION.md`](./IMPLEMENTATION.md) 第 10 章。
2. **打包流程末尾加一步自检**：用独立于写入路径的 ZIP 库打开产物、列出条目并实际解压一个，
   验证偏移修正正确。这是唯一能可靠发现 3.3.4 漏改的方法。
3. **交付前对照验收清单**：[`IMPLEMENTATION.md`](./IMPLEMENTATION.md) 第 11 章给了 6 组可勾选的验收项
   （格式一致性 / 读取 / 打包 / 安装 / 增量更新 / 安全边界）。

---

## 规范状态

| 项目              | 值                                         |
| --------------- | ----------------------------------------- |
| 规范版本            | `1.0.0`                                   |
| `formatVersion` | `1`                                       |
| 状态              | 首版发布，字段与约束已冻结                             |
| 兼容性判定依据         | `formatVersion`（主版本号），**不是** `mfpVersion` |
| JSON Schema     | draft 2020-12                             |

必需字段（不可删除/重命名/改变语义）：`mfpVersion`、`formatVersion`、`revision`、`fonts`、`checksum`。

---

## 镜像仓库

本仓库在以下平台同步维护，内容一致：

| 平台      | 地址                                                    |
| ------- | ----------------------------------------------------- |
| GitHub  | https://github.com/zyzhixi/MediaFontsPacked           |
| CNB     | https://cnb.cool/zyzhixi/MediaFontsPacked             |
| Gitee   | https://gitee.com/zyzhixi_design/MediaFontsPacked     |
| GitCode | https://gitcode.com/zyzhixi/MediaFontsPacked          |
| Codeberg | https://codeberg.org/zyzhixi_design/MediaFontsPacked |

建议以 GitHub 为主仓库发起 Issue 与 Pull Request。

---

## 参与贡献

修改规范前请先读 [`CONTRIBUTING.md`](./CONTRIBUTING.md)，其中说明了「本仓库管什么、不管什么」的边界，
以及改动字段后需要同步更新的四处位置。

## 许可证

[MIT](./LICENSE) © 2026 zyzhixi

规范文本本身以 MIT 发布，你可以自由地实现、分发基于 `.mfp` 的软件。
注意：规范允许你实现该格式，但**不授予**你对包内字体文件的任何权利——
字体的再分发受其自身许可证约束（见 `SPEC.md` 4.4.7 的 `license` 结构）。
