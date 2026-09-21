# 更新日志

本文件记录 `.mfp` 格式规范各版本的变更。格式规范的兼容性判定以 `formatVersion`（主版本号）为准，
`mfpVersion`（语义化版本）仅作信息用途，详见 [`SPEC.md`](./SPEC.md) 第 6.1 节。

## 1.0.0 — 2026-09-22

`formatVersion` = `1`

首版发布。

- 定义容器的物理字节布局：MFP Header（32 字节）+ ZIP64 归档 + 可选 MFP Footer
- 定义 `manifest.json` 的顶层结构与 `package` / `fonts` / `templates` / `conflicts` / `storage` / `checksum` / `recovery` 各子结构
- 定义增量更新的 `revision` 链语义与历史修订的存储方式（`manifests/r<revision>.json`）
- 定义条目命名规范与 `.mfp-signature` 签名条目格式
- 定义版本兼容规则、读取降级规则与兼容性矩阵
- 定义格式识别流程与读取侧完整性校验顺序
- 提供 manifest 与模板的 JSON Schema、错误码表、最小与完整示例
- 配套发布 [`IMPLEMENTATION.md`](./IMPLEMENTATION.md)（客户端实现指南）

本版本含 [`SPEC.md`](./SPEC.md) 第 10.1 节列出的 15 项设计说明，逐条记录了关键决策与理由：

| 编号 | 类型       | 议题                     |
| -- | -------- | ---------------------- |
| 1  | **关键约束** | Header 前置必须修正 ZIP 偏移（5 处字段） |
| 2  | 语义明确     | `manifestOffset` 指向 Local File Header |
| 3  | 覆盖范围     | `headerCRC32` 覆盖 28 字节 |
| 4  | 覆盖范围     | `footerCRC32` 覆盖 20 字节 |
| 5  | 语义明确     | 历史修订存于 `manifests/r<N>.json`，回溯缺失即视为链终点 |
| 6  | 约定       | 压缩方法按条目类型约定（字体 `store`、文本 `deflate`） |
| 7  | 约定       | 数据描述符的允许与读取优先级（以中央目录为准） |
| 8  | 统一       | 预览图命名规则统一 |
| 9  | 约定       | `.mfp-signature` 的内容格式 |
| 10 | 补充       | 错误码表 |
| 11 | 补充       | manifest 与模板的 JSON Schema |
| 12 | 补充       | 兼容性矩阵 |
| 13 | 补充       | 最小与完整示例 |
| 14 | 补充       | 完整性校验顺序 |
| 15 | 修正       | flags 位 3 改为 `HAS_HISTORY` |

## 版本策略

- 新增**可选**字段属于兼容变更，提升 `mfpVersion` 的次版本号，`formatVersion` 不变
- 删除或重命名**必需**字段、改变布局或约束语义，属于不兼容变更，**必须**提升 `formatVersion` 主版本号
- 必需字段清单：`mfpVersion`、`formatVersion`、`revision`、`fonts`、`checksum`
- 可选字段清单：`parentRevision`、`operation`、`changes`、`templates`、`conflicts`、`storage`、`recovery`
