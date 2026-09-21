# .mfp 格式规范 v1.0

> 多媒体字体包私有格式 · 协议层规范
> 配套实现指南见 [`IMPLEMENTATION.md`](./IMPLEMENTATION.md)

***

## 1. 文档说明

### 1.1 范围

本文档定义 `.mfp`（**M**edia **F**onts **P**acked，多媒体字体包）文件格式的**协议层**内容：

- 文件的物理字节布局
- 各字段的名称、类型、语义与约束
- 版本兼容规则与读取降级规则
- 格式识别流程
- 格式为客户端预留的扩展点

本文档**不定义**任何实现细节。以下内容属于客户端职责，不在本规范范围内：增量更新算法、校验和计算与验证、恢复记录生成与修复、预览图渲染、字体元数据解析、去重与冲突检测、字体安装与注册表写入、回收阈值的动态调整策略。

### 1.2 读者

- **格式实现者**：需要读写 `.mfp` 的客户端开发者
- **打包工具开发者**：需要生成 `.mfp` 的工具链开发者
- **生态集成者**：需要在自有软件中支持 `.mfp` 的第三方

### 1.3 术语

| 术语                  | 含义                                                       |
| ------------------- | -------------------------------------------------------- |
| **容器**              | `.mfp` 文件本身，物理上是一个带 MFP 头尾的 ZIP64 归档                     |
| **条目（Entry）**       | ZIP 归档中的一个成员，由条目名唯一标识                                    |
| **manifest**        | 容器的唯一索引文件，条目名固定为 `manifest.json`                         |
| **修订（Revision）**    | manifest 的一次版本，由 `revision` 字段标识，通过 `parentRevision` 串成链 |
| **face**            | 字体的一个"面"，即一款字重/样式。TTF/OTF 通常 1 个 face，TTC/OTC 可有多个       |
| **垃圾数据（Garbage）**   | 因增量更新而失效、但仍占据文件空间的旧条目数据                                  |
| **紧凑化（Compaction）** | 丢弃垃圾数据、完全重写容器的操作                                         |

### 1.4 关键词约定

本文档使用 RFC 2119 的关键词约定：

| 关键词               | 含义                       |
| ----------------- | ------------------------ |
| **必须（MUST）**      | 绝对要求。违反即视为不符合本规范         |
| **必须不（MUST NOT）** | 绝对禁止                     |
| **应当（SHOULD）**    | 强烈建议。存在正当理由时可例外，但需在实现中说明 |
| **可以（MAY）**       | 可选                       |

### 1.5 设计前提（重要）

本格式基于标准 ZIP64 构建。**通用压缩工具（7-Zip、WinRAR、`unzip`）可以正常解压并读出全部条目名列表。**

这是**刻意的设计选择**，而不是需要规避的缺陷：

1. **生态互操作性**：用户可以用任何熟悉的工具查看、备份、迁移包内资源，不必被单一客户端绑定
2. **可排障性**：当客户端出现问题时，可以直接用通用工具定位是容器结构问题还是客户端逻辑问题
3. **零成本迁移路径**：已有 ZIP 工具链的团队可以直接复用现有流程生成和校验容器
4. **格式识别**：文件头有固定 magic，客户端能一眼识别；`manifestOffset` 提供直达索引的快速路径

内容层面的完整性由以下机制承担：

- **`.mfp-signature`** **条目**（第 5.2 节）：允许分发方对条目签名，接收方可验证来源与完整性
- **`checksum`** **结构**（第 4.7 节）：逐条目 SHA-256，保证内容未被篡改

> 如果需求方期望的是"物理上无法解压"，本格式不适用。

***

## 2. 设计目标与边界

### 2.1 设计目标

| 目标        | 说明                                                |
| --------- | ------------------------------------------------- |
| 单文件承载多字体  | 把 TTF / OTF / TTC / OTC 等打包进一个容器，按需安装单个或全部字体      |
| 内容寻址与去重   | 字体条目以内容 SHA-256 命名，天然支持去重与完整性校验                   |
| 支持超大文件    | 强制 ZIP64，单容器可超过 4 GB                              |
| 流式读取与随机访问 | 中央目录置于文件末尾，客户端只需读索引即可按需提取单个条目                     |
| 高频增量更新    | 通过 ZIP 追加语义 + revision 链，新增/删除字体只重写中央目录，不触碰已有条目数据 |
| 跨平台可移植    | 格式本身不含平台相关内容，安装策略由客户端决定                           |
| 向后兼容      | 新增可选字段不影响旧客户端读取                                   |

### 2.2 职责边界

| 层面   | 格式（本规范）负责                                 | 客户端负责                  |
| ---- | ----------------------------------------- | ---------------------- |
| 布局   | 头尾结构、字段偏移、字节序                             | 读写实现                   |
| 索引   | manifest 字段定义与语义                          | manifest 的生成与解析        |
| 增量更新 | `revision` 链、`changes` 字段的语义              | 截断 CD、追加条目、重写 CD 的具体算法 |
| 校验和  | `checksum` 结构                             | 哈希计算与验证时机              |
| 恢复记录 | Footer 位置、`recoveryLength`、`coveredRange` | 生成算法、修复逻辑              |
| 预览   | 模板数据结构、变量列表                               | 渲染引擎、字体加载、回退链          |
| 元数据  | `metadata` 字段定义                           | 用字体引擎解析字体              |
| 冲突   | `conflicts` 记录结构                          | 检测算法、UI 呈现             |
| 安装   | `installMode` 字段语义                        | 平台 API、注册表写入、提权        |
| 回收   | `storage` 结构、阈值区间                         | 动态调整策略、紧凑化执行           |

***

## 3. 文件物理布局

### 3.1 总体结构

```
偏移 0
┌────────────────────────────────────────────────┐
│  MFP Header（固定 32 字节）                     │
├────────────────────────────────────────────────┤
│  ZIP64 归档负载                                 │
│  ┌──────────────────────────────────────────┐  │
│  │ 条目区                                    │  │
│  │   Local File Header + 数据（逐条）        │  │
│  │   manifest.json 必须是第一个条目          │  │
│  ├──────────────────────────────────────────┤  │
│  │ Central Directory（明文）                 │  │
│  ├──────────────────────────────────────────┤  │
│  │ ZIP64 End of Central Directory Record     │  │
│  ├──────────────────────────────────────────┤  │
│  │ ZIP64 End of Central Directory Locator    │  │
│  ├──────────────────────────────────────────┤  │
│  │ End of Central Directory Record           │  │
│  └──────────────────────────────────────────┘  │
├────────────────────────────────────────────────┤
│  MFP Footer（可变长度，可选）                   │
│  ┌──────────────────────────────────────────┐  │
│  │ Recovery Record（长度由 recoveryLength）  │  │
│  ├──────────────────────────────────────────┤  │
│  │ Footer Descriptor（固定 24 字节）         │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘ 文件末尾
```

### 3.2 MFP Header

固定 32 字节，**必须**位于文件起始偏移 0。

| 偏移 | 长度 | 字段               | 类型     | 说明                                                |
| -- | -- | ---------------- | ------ | ------------------------------------------------- |
| 0  | 4  | `magic`          | bytes  | 固定 `0x4D 0x46 0x50 0x01`，即 ASCII `"MFP"` + `0x01` |
| 4  | 2  | `formatVersion`  | uint16 | 格式主版本号，当前为 `1`                                    |
| 6  | 2  | `flags`          | uint16 | 位标志，见 3.2.1                                       |
| 8  | 8  | `createdAt`      | uint64 | 容器首次创建时间，Unix 毫秒时间戳                               |
| 16 | 4  | `headerCRC32`    | uint32 | 头部校验值，覆盖范围见 3.2.2                                 |
| 20 | 4  | `reserved`       | uint32 | 保留，**必须**填 `0`                                    |
| 24 | 8  | `manifestOffset` | uint64 | manifest 条目定位，语义见 3.2.3                           |

#### 3.2.1 flags 位定义

| 位    | 名称             | 说明                         |
| ---- | -------------- | -------------------------- |
| 0    | 保留             | **必须**填 `0`                |
| 1    | `HAS_RECOVERY` | 置 1 表示文件尾部存在恢复记录           |
| 2    | `HAS_FOOTER`   | 置 1 表示存在 Footer Descriptor |
| 3    | `HAS_HISTORY`  | 置 1 表示容器内保留了历史修订（见 4.2.3）  |
| 4–15 | 保留             | **必须**填 0                  |

> **设计说明 15｜flags 位 3 承载实际信息**
> 位 3 定义为 `HAS_HISTORY`（是否保留历史修订），而非"中央目录置于文件末尾"之类的结构描述。
> **理由**："中央目录置于末尾"是本格式的**强制约束**（见第 7 节约束 2），若把它做成标志位则该位恒为 1、不携带任何信息。`HAS_HISTORY` 对读取侧有实际影响——客户端可据此决定是否需要尝试回溯 `parentRevision` 链。
> 位 0 保留未定义，为将来的格式演进留出空间；不占用位 1–3 以免打乱已定义的语义。

#### 3.2.2 headerCRC32 覆盖范围

`headerCRC32` **必须**覆盖头部中除自身 4 字节（偏移 16–19）以外的全部 28 字节，即：偏移 0–15 与偏移 20–31 按顺序拼接后计算 CRC32。

- CRC32 参数：多项式 `0xEDB88320`（反射）、初值 `0xFFFFFFFF`、结果取反（即 ZIP 使用的标准 CRC-32）
- 计算时偏移 16–19 按全 `0x00` 参与

> **设计说明 3｜headerCRC32 必须覆盖 28 字节**
> 若只覆盖前 16 字节，则 `reserved` 与 `manifestOffset` 不受任何保护。`manifestOffset` 被篡改会导致客户端读取错误位置、甚至读到攻击者构造的数据，因此必须纳入校验。
> **理由**：头部的每一个可影响读取路径的字段都应当被校验覆盖。

#### 3.2.3 manifestOffset 语义

`manifestOffset` 表示 **`manifest.json`** **条目的 Local File Header 起始位置在文件中的绝对偏移**（从文件偏移 0 起算，**已包含 32 字节 MFP Header**）。

- 指向 Local File Header 的签名 `0x04034B50`，而非条目数据区
- **不**指向 ZIP 内部坐标；读取时若需换算为 ZIP 内部坐标，减去 32
- 每次容器被修改后（增量更新、紧凑化）**必须**重写该字段并重算 `headerCRC32`

> **设计说明 2｜manifestOffset 的精确语义**
> 必须明确指向 Local File Header 而非数据区，并规定增量更新后必须重写。
> **理由**：这两种歧义在实现时都会直接导致读取失败——指向数据区会让读取方在错误位置寻找签名；不重写会让 `manifestOffset` 指向已被截断的旧位置。

### 3.3 ZIP 布局约定

#### 3.3.1 强制 ZIP64

容器**必须**使用 ZIP64 扩展，即使总大小小于 4 GB：

- 中央目录条目中的偏移与大小字段**应当**使用 ZIP64 Extended Information Extra Field（header ID `0x0001`）
- ZIP64 End of Central Directory Record（签名 `0x06064B50`）与 ZIP64 End of Central Directory Locator（签名 `0x07064B50`）**必须**存在
- 标准 End of Central Directory Record（签名 `0x06054B50`）**必须**存在，作为 ZIP64 结构的向后兼容入口

强制 ZIP64 的理由：避免"文件在 4 GB 边界附近跨过阈值"导致的结构突变，也让客户端只需实现一条读取路径。

#### 3.3.2 中央目录位置

条目数据在前、中央目录在后、ZIP64 EOCD / Locator / EOCD 收尾。**必须**保持这一顺序，它是流式读取与增量追加的前提。

#### 3.3.3 中央目录明文

**中央目录与 EOCD 结构始终明文存储。**

ZIP 规范中中央目录本身不参与任何内容变换，因此通用工具无需任何额外信息即可读出全部条目名列表（例如 `fonts/a3f8...ttf`、`previews/a3f8....preview.png`）。

这是 1.5 节所述设计选择的直接体现：条目名的可见性换来的是可排障性与生态互操作性。需要隐藏条目名的场景不在本格式的适用范围内。

#### 3.3.4 偏移修正（关键约束）

MFP Header 位于 ZIP 数据之前，这使 ZIP 内部所有偏移相对"纯 ZIP 文件"整体后移 32 字节。而 ZIP 中央目录中记录的 *relative offset of local header* 是**相对文件起始的绝对偏移**，因此：

> **写入 MFP Header 后，必须对 ZIP 结构做如下修正：**

| 序号 | 需要修正的字段                                                                        | 修正方式    |
| -- | ------------------------------------------------------------------------------ | ------- |
| 1  | 每个中央目录条目的 *relative offset of local header*（4 字节）                              | `+= 32` |
| 2  | 中央目录条目 ZIP64 扩展字段中的 *local header offset*（若存在）                                 | `+= 32` |
| 3  | End of Central Directory Record 的 *offset of start of central directory*       | `+= 32` |
| 4  | ZIP64 End of Central Directory Record 的 *offset of start of central directory* | `+= 32` |
| 5  | ZIP64 End of Central Directory Locator 的 *relative offset of ZIP64 EOCD*       | `+= 32` |

**不**需要修正的字段：条目数、中央目录大小、各条目的压缩/未压缩大小、CRC32。

修正完成后，通用 ZIP 工具按中央目录寻址能落到正确的 Local File Header，**正常解压**。

读取侧对应地：把 ZIP 内部坐标换算为文件坐标时 `+= 32`；把文件坐标换算为 ZIP 内部坐标时 `-= 32`。

> **设计说明 1｜Header 前置必须修正 ZIP 偏移（最关键约束）**
> 在 ZIP 数据之前插入 32 字节 Header 后，**必须**同步修正上表 5 处字段。
> **理由**：ZIP 中央目录里记录的 *relative offset of local header* 是相对**文件起始**的绝对偏移。插入 32 字节后这些偏移全部错位，通用工具按中央目录寻址会落到错误位置，**解压直接失败**——而"标准 ZIP 工具仍可解压"是本格式明确的设计前提（1.5 节）。这是实现者最容易踩的坑，因为 MFP 客户端自己可能仍能正常读取（它走 `manifestOffset` 而非中央目录），症状极其隐蔽。

**实现建议**：先用流式写入库正常生成 ZIP，再把 32 字节 Header 前置拼接，最后从文件尾部向前搜索 EOCD（`0x06054B50`）、ZIP64 EOCD Locator（`0x07064B50`）、ZIP64 EOCD（`0x06064B50`），按上表逐字段修补。

**已评估并否决的替代方案**：

| 方案                           | 否决理由                        |
| ---------------------------- | --------------------------- |
| 把 Header 作为 ZIP 的第一个条目       | 无法通过文件头 4 字节快速识别格式，违背格式识别需求 |
| 把 magic 放进 EOCD 的 comment 字段 | 同上；且 comment 长度受限、易被工具重写    |
| 让写入库直接在偏移 32 处开始写            | 主流流式 ZIP 库不支持指定起始偏移         |

### 3.4 MFP Footer

Footer 位于 ZIP EOCD 之后，承载格式级扩展数据。`HAS_FOOTER` 置 1 时**必须**存在。

```
┌────────────────────────────────────────────────┐
│  Recovery Record（长度 = recoveryLength）       │
├────────────────────────────────────────────────┤
│  Footer Descriptor（固定 24 字节）              │
│  ├─ footerMagic    4B   固定 "MFP\xFF"          │
│  ├─ recoveryLength 8B   uint64，恢复记录字节数  │
│  ├─ footerCRC32    4B   uint32，校验值          │
│  └─ reserved       8B   保留，必须填 0          │
└────────────────────────────────────────────────┘
```

| 偏移（相对 Descriptor 起始） | 长度 | 字段               | 类型     | 说明                                                |
| -------------------- | -- | ---------------- | ------ | ------------------------------------------------- |
| 0                    | 4  | `footerMagic`    | bytes  | 固定 `0x4D 0x46 0x50 0xFF`，即 ASCII `"MFP"` + `0xFF` |
| 4                    | 8  | `recoveryLength` | uint64 | 恢复记录长度；`0` 表示无恢复记录                                |
| 12                   | 4  | `footerCRC32`    | uint32 | 校验值，覆盖范围见下                                        |
| 16                   | 8  | `reserved`       | uint64 | 保留，**必须**填 `0`                                    |

**footerCRC32 覆盖范围**：**必须**覆盖 `footerMagic`（4 字节）+ `recoveryLength`（8 字节）+ `reserved`（8 字节），共 20 字节，按偏移 0–11 与 16–23 的顺序拼接。`footerCRC32` 自身（偏移 12–15）不参与，计算时按全 `0x00` 处理。CRC32 参数同 3.2.2。

> **设计说明 4｜footerCRC32 覆盖 20 字节**
> `footerCRC32` 必须覆盖除自身以外的全部字段（`footerMagic` + `recoveryLength` + `reserved`）。
> **理由**：`recoveryLength` 直接决定恢复记录的读取范围，若不受校验，被篡改后客户端会读取错误的字节区间，可能把无关数据当作恢复记录处理。

**恢复记录的生成算法与内容格式由客户端定义。** 本规范只规定：

- 位置：紧邻 Footer Descriptor 之前
- 长度：由 `recoveryLength` 精确指明
- 覆盖范围：由 manifest 的 `recovery.coveredRange` 声明（见 4.9），且**必须**与 `recoveryLength` 所覆盖的实际字节范围一致

**读取规则**：先定位 ZIP EOCD，再检查其**紧接着**的位置是否为 `"MFP\xFF"`。是则解析 Footer Descriptor；否则视为无 Footer。若 `HAS_FOOTER` 置 1 但 EOCD 后未找到 `footerMagic`，客户端**应当**报告 `E_HEADER_CRC` 或 `E_ZIP_STRUCTURE`，而非静默忽略。

### 3.5 字节序与编码约定

**字节序**：MFP Header 与 Footer Descriptor 中的所有多字节整数**必须**使用**小端序（little-endian）**。这与 ZIP 格式自身的字节序一致，避免读写时反复切换。

**数值编码**：

| 类型       | 字节数 | 说明                                                   |
| -------- | --- | ---------------------------------------------------- |
| `uint16` | 2   | 小端                                                   |
| `uint32` | 4   | 小端                                                   |
| `uint64` | 8   | 小端；超过 `Number.MAX_SAFE_INTEGER` 的值**必须**用 64 位整数类型处理 |

**CRC32 参数**：多项式 `0xEDB88320`（反射），初值 `0xFFFFFFFF`，输出取反。与 ZIP 条目 CRC32 使用同一算法。

**magic 常量表**：

| 常量                    | 值（十六进制）       | ASCII        | 位置                     |
| --------------------- | ------------- | ------------ | ---------------------- |
| `MFP_MAGIC`           | `4D 46 50 01` | `MFP\x01`    | 文件偏移 0                 |
| `MFP_FOOTER_MAGIC`    | `4D 46 50 FF` | `MFP\xFF`    | Footer Descriptor 偏移 0 |
| `ZIP_LOCAL_HEADER`    | `50 4B 03 04` | `PK\x03\x04` | 各条目起始                  |
| `ZIP_CENTRAL_HEADER`  | `50 4B 01 02` | `PK\x01\x02` | 中央目录条目                 |
| `ZIP_EOCD`            | `50 4B 05 06` | `PK\x05\x06` | ZIP 末尾                 |
| `ZIP64_EOCD`          | `50 4B 06 06` | `PK\x06\x06` | ZIP64 EOCD             |
| `ZIP64_EOCD_LOCATOR`  | `50 4B 06 07` | `PK\x06\x07` | ZIP64 EOCD Locator     |
| `ZIP_DATA_DESCRIPTOR` | `50 4B 07 08` | `PK\x07\x08` | 数据描述符（可选）              |

**文件名编码**：条目名**必须**使用 UTF-8 编码，并在 ZIP 通用位标志（general purpose bit flag）中置位 **bit 11（EFS，UTF-8 标志）**。条目名**必须不**包含反斜杠 `\`，路径分隔符统一使用正斜杠 `/`。

**时间戳**：MFP Header 的 `createdAt` 为 Unix 毫秒时间戳。ZIP 条目自身的 MS-DOS 时间字段仅作兼容用途，客户端**应当**以 manifest 中的 ISO 8601 时间为准。

***

## 4. manifest.json

manifest 是容器的唯一索引文件，条目名固定为 `manifest.json`，**必须**是 ZIP 的**第一个条目**（便于流式读取时优先获取索引）。

### 4.1 顶层结构

```json
{
  "mfpVersion": "1.0.0",
  "formatVersion": 1,
  "revision": 42,
  "parentRevision": 41,
  "createdAt": "2026-09-20T12:00:00Z",
  "updatedAt": "2026-09-20T15:30:00Z",
  "operation": "addFont",
  "changes": { "...": "..." },
  "package": { "...": "..." },
  "fonts": [ "..." ],
  "templates": [ "..." ],
  "conflicts": { "...": "..." },
  "storage": { "...": "..." },
  "checksum": { "...": "..." },
  "recovery": { "...": "..." }
}
```

| 字段               | 类型             | 必需 | 说明                                                 |
| ---------------- | -------------- | -- | -------------------------------------------------- |
| `mfpVersion`     | string         | 必须 | 本规范版本，语义化版本，当前 `"1.0.0"`                           |
| `formatVersion`  | uint32         | 必须 | 与 Header 的 `formatVersion` 一致，冗余存储以便 manifest 独立自洽 |
| `revision`       | uint32         | 必须 | 当前修订号，从 `1` 开始，每次修改 `+1`                           |
| `parentRevision` | uint32 \| null | 可以 | 父修订号。初始 manifest 为 `null`                          |
| `createdAt`      | string         | 必须 | ISO 8601 UTC 时间，容器首次创建时间                           |
| `updatedAt`      | string         | 必须 | ISO 8601 UTC 时间，本次修订时间                             |
| `operation`      | string         | 可以 | 本次操作类型，见 4.2.2                                     |
| `changes`        | object         | 可以 | 本次变更摘要，见 4.2.2                                     |
| `package`        | object         | 必须 | 包级信息，见 4.3                                         |
| `fonts`          | array          | 必须 | 字体条目数组，至少一个元素，见 4.4                                |
| `templates`      | array          | 可以 | 预览模板列表，见 4.5                                       |
| `conflicts`      | object         | 可以 | 冲突记录，见 4.8                                         |
| `storage`        | object         | 可以 | 存储与回收信息，见 4.6                                      |
| `checksum`       | object         | 必须 | 校验和，见 4.7                                          |
| `recovery`       | object         | 可以 | 恢复记录信息，见 4.9                                       |

### 4.2 增量更新字段

#### 4.2.1 revision 链

- `revision` **必须**单调递增，初始值为 `1`
- 每次对容器的修改（新增字体、删除字体、更新元数据、紧凑化）**必须**生成一个新的 manifest，`revision` 加 1，`parentRevision` 指向被替换的那一版
- 客户端读取时从最新 manifest 出发，沿 `parentRevision` 回溯，重建完整索引
- 回溯到 `parentRevision: null` 即到达链起点

#### 4.2.2 operation 与 changes

`operation` 取值：

| 值                | 含义               |
| ---------------- | ---------------- |
| `create`         | 容器首次创建           |
| `addFont`        | 新增字体             |
| `removeFont`     | 删除字体             |
| `updateMetadata` | 更新包级或字体级元数据      |
| `compact`        | 紧凑化（丢弃垃圾数据后完全重写） |

`changes` 结构：

```json
{
  "changes": {
    "added": ["font-005"],
    "removed": ["font-002"],
    "modified": ["fonts/font-001", "templates/default-v1"],
    "compacted": false
  }
}
```

| 字段          | 类型        | 说明              |
| ----------- | --------- | --------------- |
| `added`     | string\[] | 本次新增的字体 id 列表   |
| `removed`   | string\[] | 本次移除的字体 id 列表   |
| `modified`  | string\[] | 本次修改的条目名或资源标识列表 |
| `compacted` | boolean   | 本次是否为紧凑化操作      |

`changes` 是**信息性**字段：客户端**可以**用它做差异同步或审计，但**必须不**依赖它重建索引——重建**必须**以完整的 `fonts` 数组为准。

#### 4.2.3 历史修订的存储

**`manifest.json`** **恒为最新修订，且必须是 ZIP 的第一个条目。**

历史修订的保留是**可选**的：

- 保留历史时，每一版**必须**以独立条目存储，条目名为 `manifests/r<revision>.json`（例如 `manifests/r41.json`）
- 保留历史时，Header 的 `HAS_HISTORY` 位置 1
- 客户端**可以**自行决定保留多少个历史修订，本规范不做限制

**回溯的降级规则**：回溯过程中若发现某个 `parentRevision` 对应的历史条目不存在，客户端**必须**将其视为链的终点（即认为已到达可回溯的最早版本），**必须不**因此报错。原因：历史保留是可选的，缺失是正常状态而非损坏。

> **设计说明 5｜历史修订的存储位置与降级行为**
> 规定 `manifest.json` 恒为最新且必须是第一个条目，历史修订存为 `manifests/r<revision>.json`；回溯时找不到历史条目即视为到达链起点，**不报错**。
> **理由**：若把 manifest 设计成"同名条目被覆盖"，历史修订就全部丢失，`parentRevision` 回溯无从谈起；反之若不明确降级行为，实现者会把"历史未保留"误判为"容器损坏"而拒绝读取。

### 4.3 package 结构

```json
{
  "package": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "My Font Pack",
    "description": "A collection of display fonts",
    "version": "2.1.0",
    "cover": "assets/cover.png",
    "screenshots": ["assets/screenshots/01.png"],
    "icon": "assets/icons/icon.png",
    "eula": "licenses/eula.txt",
    "allowRedistribution": true,
    "tags": ["display", "serif"]
  }
}
```

| 字段                    | 类型        | 必需 | 说明                      |
| --------------------- | --------- | -- | ----------------------- |
| `id`                  | string    | 必须 | 包的唯一标识，**应当**使用 UUID v4 |
| `name`                | string    | 必须 | 包名                      |
| `description`         | string    | 可以 | 描述                      |
| `version`             | string    | 必须 | 包版本，语义化版本               |
| `cover`               | string    | 可以 | 封面图条目名                  |
| `screenshots`         | string\[] | 可以 | 截图条目名列表                 |
| `icon`                | string    | 可以 | 图标条目名                   |
| `eula`                | string    | 可以 | 包级 EULA 文本条目名           |
| `allowRedistribution` | boolean   | 可以 | 是否允许再分发，缺省视为 `false`    |
| `tags`                | string\[] | 可以 | 标签                      |

### 4.4 fonts 条目结构

#### 4.4.1 单字体（TTF / OTF / WOFF / WOFF2）

```json
{
  "id": "font-001",
  "hash": "sha256:a3f8c2d1e5b7...",
  "file": "fonts/a3f8c2d1e5b7....ttf",
  "originalName": "MyFont-Regular.ttf",
  "format": "ttf",
  "isCollection": false,
  "faceCount": 1,
  "installMode": "singleFace",
  "metadata": { "...": "..." },
  "preview": { "...": "..." },
  "license": { "...": "..." }
}
```

#### 4.4.2 集合字体（TTC / OTC）

```json
{
  "id": "font-001",
  "hash": "sha256:a3f8c2d1...",
  "file": "fonts/a3f8c2d1....ttc",
  "originalName": "MyFontCollection.ttc",
  "format": "ttc",
  "isCollection": true,
  "faceCount": 4,
  "installMode": "allFaces",
  "faces": [
    {
      "faceIndex": 0,
      "postscriptName": "MyFont-Regular",
      "familyName": "My Font",
      "subfamilyName": "Regular",
      "version": "1.000",
      "characterSet": { "unicodeRanges": ["U+0020-U+007E"], "glyphCount": 567 },
      "variationAxes": null,
      "colorTables": []
    }
  ],
  "metadata": { "...": "..." },
  "preview": { "...": "..." }
}
```

#### 4.4.3 字段说明

| 字段             | 类型      | 必需 | 说明                                               |
| -------------- | ------- | -- | ------------------------------------------------ |
| `id`           | string  | 必须 | 包内唯一标识，**应当**使用 `font-<序号>` 形式                   |
| `hash`         | string  | 必须 | 条目内容的 SHA-256，格式 `sha256:<64 位小写十六进制>`           |
| `file`         | string  | 必须 | 字体条目名，**必须**符合第 5 节的命名规范                         |
| `originalName` | string  | 可以 | 打包前的原始文件名，供用户识别                                  |
| `format`       | string  | 必须 | `ttf` / `otf` / `ttc` / `otc` / `woff` / `woff2` |
| `isCollection` | boolean | 必须 | 是否为集合字体                                          |
| `faceCount`    | uint32  | 必须 | face 数量，单字体为 `1`                                 |
| `installMode`  | string  | 必须 | `singleFace` 或 `allFaces`，见 4.4.4                |
| `faces`        | array   | 可以 | face 明细；`isCollection` 为 `true` 时**应当**提供        |
| `metadata`     | object  | 必须 | 字体元数据，见 4.4.5                                    |
| `preview`      | object  | 可以 | 预览信息，见 4.4.6                                     |
| `license`      | object  | 可以 | 许可证信息，见 4.4.7                                    |

#### 4.4.4 installMode 语义

| 值            | 适用格式                     | 含义                              |
| ------------ | ------------------------ | ------------------------------- |
| `singleFace` | TTF / OTF / WOFF / WOFF2 | 安装单个字体文件                        |
| `allFaces`   | TTC / OTC                | 安装集合内的**全部** face（集合文件整体安装，不拆分） |

**约束**：`isCollection` 为 `true` 时 `installMode` **必须**为 `allFaces`。本格式 v1 **不定义**"只安装集合中某一个 face"的模式；需要该能力时**必须**提升 `formatVersion` 主版本号。

#### 4.4.5 metadata 结构

单字体与集合字体共用同一结构。集合字体时，`metadata` 记录集合的汇总信息（**应当**取第一个 face 或最具代表性的 face）。

```json
{
  "metadata": {
    "familyName": "My Font",
    "subfamilyName": "Regular",
    "postscriptName": "MyFont-Regular",
    "fullName": "My Font Regular",
    "version": "1.000",
    "copyright": "Copyright 2026",
    "designer": "Jane Doe",
    "designerURL": "https://example.com",
    "license": "OFL-1.1",
    "licenseURL": "https://scripts.sil.org/OFL",
    "characterSet": {
      "unicodeRanges": ["U+0020-U+007E", "U+00A0-U+00FF"],
      "glyphCount": 1234
    },
    "variationAxes": {
      "wght": { "name": "Weight", "min": 100, "default": 400, "max": 900 }
    },
    "colorTables": ["COLR", "CPAL"],
    "availableFormats": ["ttf", "woff2"],
    "woff2File": "fonts/a3f8c2d1....woff2"
  }
}
```

| 字段                           | 类型             | 必需 | 说明                                       |
| ---------------------------- | -------------- | -- | ---------------------------------------- |
| `familyName`                 | string         | 必须 | 字体家族名                                    |
| `subfamilyName`              | string         | 必须 | 子家族名（字重/样式）                              |
| `postscriptName`             | string         | 必须 | PostScript 名称                            |
| `fullName`                   | string         | 必须 | 完整名称                                     |
| `version`                    | string         | 可以 | 字体版本                                     |
| `copyright`                  | string         | 可以 | 版权声明                                     |
| `designer`                   | string         | 可以 | 设计师                                      |
| `designerURL`                | string         | 可以 | 设计师网址                                    |
| `license`                    | string         | 可以 | 许可证标识（如 `OFL-1.1`、`Apache-2.0`）          |
| `licenseURL`                 | string         | 可以 | 许可证网址                                    |
| `characterSet`               | object         | 可以 | 字符集信息                                    |
| `characterSet.unicodeRanges` | string\[]      | 可以 | Unicode 范围，格式 `U+XXXX` 或 `U+XXXX-U+YYYY` |
| `characterSet.glyphCount`    | uint32         | 可以 | 字形总数                                     |
| `variationAxes`              | object \| null | 可以 | 可变轴；`null` 表示静态字体                        |
| `colorTables`                | string\[]      | 可以 | 颜色表标识（如 `COLR`、`CPAL`、`sbix`）            |
| `availableFormats`           | string\[]      | 可以 | 同一字体可用的全部格式                              |
| `woff2File`                  | string         | 可以 | 同字体的 WOFF2 条目名（若打包了多格式）                  |

#### 4.4.6 preview 结构

```json
{
  "preview": {
    "image": "previews/a3f8c2d1....preview.png",
    "templateId": "default-v1",
    "generatedText": "AaBbCc 你好世界 0123",
    "source": "generated"
  }
}
```

| 字段              | 类型     | 必需 | 说明             |
| --------------- | ------ | -- | -------------- |
| `image`         | string | 可以 | 预览图条目名         |
| `templateId`    | string | 可以 | 生成该预览图所用的模板 id |
| `generatedText` | string | 可以 | 生成预览时使用的文字     |
| `source`        | string | 可以 | 预览来源，取值见下      |

`source` 取值：

| 值           | 含义               |
| ----------- | ---------------- |
| `user`      | 打包时由用户提供的预览图     |
| `generated` | 打包时自动渲染生成        |
| `embedded`  | 取自字体文件内嵌的预览资源    |
| `fallback`  | 无预览图，读取时由客户端回退渲染 |

客户端**应当**在 `source` 为 `fallback` 或 `image` 缺失时，于读取阶段自行生成预览。

#### 4.4.7 license 结构

```json
{
  "license": {
    "type": "OFL-1.1",
    "file": "licenses/a3f8c2d1....license.txt",
    "allowRedistribution": true,
    "allowCommercial": true,
    "allowModification": true
  }
}
```

| 字段                    | 类型      | 必需 | 说明       |
| --------------------- | ------- | -- | -------- |
| `type`                | string  | 可以 | 许可证标识    |
| `file`                | string  | 可以 | 许可证文本条目名 |
| `allowRedistribution` | boolean | 可以 | 是否允许再分发  |
| `allowCommercial`     | boolean | 可以 | 是否允许商用   |
| `allowModification`   | boolean | 可以 | 是否允许修改   |

**约束**：`allowRedistribution` 缺省**必须**视为 `false`。客户端在导出或再分发字体前**应当**检查该字段。

### 4.5 templates 结构

本格式只定义模板的**数据结构**，渲染由客户端实现。**只支持单字体预览。**

```json
{
  "templates": [
    {
      "id": "default-v1",
      "name": "默认预览模板",
      "isDefault": true,
      "file": "templates/default-v1.json"
    }
  ]
}
```

| 字段          | 类型      | 必需 | 说明                              |
| ----------- | ------- | -- | ------------------------------- |
| `id`        | string  | 必须 | 模板标识，包内唯一                       |
| `name`      | string  | 必须 | 模板名称                            |
| `isDefault` | boolean | 可以 | 是否为默认模板；一个包内**至多一个**模板可置 `true` |
| `file`      | string  | 必须 | 模板 JSON 条目名                     |

#### 4.5.1 模板文件内容

```json
{
  "templateId": "default-v1",
  "name": "默认预览模板",
  "canvas": {
    "width": 600,
    "height": 200,
    "background": "#FFFFFF"
  },
  "textLayers": [
    {
      "id": "sample",
      "text": "AaBbCc 你好世界 0123",
      "fontSize": 36,
      "color": "#333333",
      "x": 30,
      "y": 100,
      "align": "left",
      "baseline": "middle"
    },
    {
      "id": "family",
      "text": "{{familyName}}",
      "fontSize": 14,
      "color": "#999999",
      "x": 30,
      "y": 170,
      "align": "left"
    }
  ],
  "decorations": [
    {
      "type": "rect",
      "x": 0, "y": 0, "width": 600, "height": 4,
      "fill": "#4A90D9"
    }
  ],
  "fallback": {
    "text": "{{fullName}}",
    "fontSize": 24,
    "color": "#333333"
  }
}
```

**canvas**：`width`、`height` 为像素，**必须**为正整数且不超过 `4096`；`background` 为 `#RRGGBB` 或 `#RRGGBBAA`。

**textLayers**：数组元素字段说明——

| 字段         | 类型     | 必需 | 说明                                                         |
| ---------- | ------ | -- | ---------------------------------------------------------- |
| `id`       | string | 可以 | 图层标识                                                       |
| `text`     | string | 必须 | 文本内容，可含模板变量                                                |
| `fontSize` | number | 必须 | 字号（像素）                                                     |
| `color`    | string | 可以 | 文本颜色，缺省 `#333333`                                          |
| `x`        | number | 必须 | 锚点 X 坐标（像素）                                                |
| `y`        | number | 必须 | 锚点 Y 坐标（像素）                                                |
| `align`    | string | 可以 | `left` / `center` / `right`，缺省 `left`                      |
| `baseline` | string | 可以 | `top` / `middle` / `bottom` / `alphabetic`，缺省 `alphabetic` |

**decorations**：数组元素字段说明——

| 字段                 | 类型     | 必需 | 说明           |
| ------------------ | ------ | -- | ------------ |
| `type`             | string | 必须 | 目前只定义 `rect` |
| `x` / `y`          | number | 必须 | 左上角坐标        |
| `width` / `height` | number | 必须 | 尺寸           |
| `fill`             | string | 可以 | 填充色          |

**fallback**：渲染失败时的纯文本回退配置，字段为 `text`、`fontSize`、`color`。

#### 4.5.2 模板变量

| 变量                   | 来源                                 |
| -------------------- | ---------------------------------- |
| `{{familyName}}`     | `metadata.familyName`              |
| `{{subfamilyName}}`  | `metadata.subfamilyName`           |
| `{{fullName}}`       | `metadata.fullName`                |
| `{{postscriptName}}` | `metadata.postscriptName`          |
| `{{version}}`        | `metadata.version`                 |
| `{{copyright}}`      | `metadata.copyright`               |
| `{{designer}}`       | `metadata.designer`                |
| `{{glyphCount}}`     | `metadata.characterSet.glyphCount` |

未知变量**必须**替换为空字符串，**必须不**导致渲染失败。

#### 4.5.3 模板约束

- 模板**必须**只描述单字体渲染。若模板中出现多字体混排的定义，客户端**应当**降级为单字体渲染，或判定该模板无效（`E_TEMPLATE_INVALID`）
- 模板变量**必须**在渲染时按 4.5.2 替换，客户端**必须不**执行任意代码或表达式求值

### 4.6 storage 结构

```json
{
  "storage": {
    "garbageBytes": 52428800,
    "garbageRatio": 0.12,
    "compactionThreshold": 0.2,
    "lastCompactionRevision": 30,
    "entryCount": 47,
    "liveEntryCount": 42
  }
}
```

| 字段                       | 类型     | 必需 | 说明                                 |
| ------------------------ | ------ | -- | ---------------------------------- |
| `garbageBytes`           | uint64 | 可以 | 容器内已失效条目占用的字节数                     |
| `garbageRatio`           | number | 可以 | `garbageBytes / 容器总大小`，取值 `[0, 1]` |
| `compactionThreshold`    | number | 可以 | 触发紧凑化的阈值                           |
| `lastCompactionRevision` | uint32 | 可以 | 上次紧凑化对应的 `revision`                |
| `entryCount`             | uint32 | 可以 | 中央目录中的条目总数（含失效条目）                  |
| `liveEntryCount`         | uint32 | 可以 | 有效条目数                              |

**动态调整策略由客户端实现**，本规范只规定字段含义与**区间约束**：

> `compactionThreshold` **必须**落在 `[0.05, 0.50]` 闭区间内。

低于 `0.05` 会导致过于频繁的全量重写，高于 `0.50` 会让垃圾数据占比过高、容器体积失控。客户端**可以**基于以下因素在区间内动态调整：

- 容器总大小（大文件可容忍更高的垃圾比例）
- 增量更新频率（高频更新宜降低阈值）
- 剩余磁盘空间（空间紧张时宜降低阈值）
- 上次紧凑化耗时（耗时过长时宜提高阈值）

`storage` 缺失时，客户端**必须不**执行自动紧凑化，改由用户手动触发。

### 4.7 checksum 结构

```json
{
  "checksum": {
    "algorithm": "sha256",
    "manifestHash": "sha256:...",
    "entries": {
      "fonts/a3f8c2d1....ttf": "sha256:...",
      "previews/a3f8c2d1....preview.png": "sha256:..."
    }
  }
}
```

| 字段             | 类型     | 必需 | 说明             |
| -------------- | ------ | -- | -------------- |
| `algorithm`    | string | 必须 | 目前只定义 `sha256` |
| `manifestHash` | string | 必须 | manifest 自身的哈希 |
| `entries`      | object | 必须 | 条目名 → 内容哈希的映射  |

**约束**：

- `entries` **必须**包含除 `manifest.json` 自身以外的**全部**条目
- `manifestHash` 是 manifest 文件的哈希，计算时 `manifestHash` 字段**必须**置为空字符串 `""`（避免自引用）
- 客户端在读取时**可以**选择验证；验证失败**应当**报告 `E_CHECKSUM_MISMATCH`

### 4.8 conflicts 结构

```json
{
  "conflicts": {
    "duplicates": [
      { "fontId": "font-002", "duplicateOf": "font-001", "type": "exact" }
    ],
    "versionConflicts": [
      { "fontId": "font-003", "conflictsWith": "font-001", "type": "version" }
    ],
    "nameConflicts": [
      { "fontId": "font-004", "conflictsWith": "font-001", "type": "name" }
    ],
    "formatDuplicates": [
      { "fontId": "font-005", "duplicateOf": "font-001", "type": "format" }
    ]
  }
}
```

| 分组                 | `type`    | 判定依据                          | 处理建议            |
| ------------------ | --------- | ----------------------------- | --------------- |
| `duplicates`       | `exact`   | 文件 SHA-256 相同                 | 完全重复，**应当**跳过后者 |
| `versionConflicts` | `version` | PostScript 名相同、版本不同           | 交由用户选择保留哪一版     |
| `nameConflicts`    | `name`    | 家族名 + 子家族名相同，但 PostScript 名不同 | 提示用户可能混淆        |
| `formatDuplicates` | `format`  | 同一字体的不同格式（如 TTF 与 OTF）        | **应当**只保留一种     |

**约束**：本格式只规定冲突的**记录方式**，检测逻辑由客户端实现。客户端**可以**自行决定如何呈现与处置；`conflicts` 缺失时**必须**跳过冲突提示，**必须不**因此拒绝读取。

### 4.9 recovery 结构

```json
{
  "recovery": {
    "present": true,
    "method": "client-defined",
    "coveredRange": "central-directory+manifest",
    "length": 1048576
  }
}
```

| 字段             | 类型      | 必需 | 说明                                |
| -------------- | ------- | -- | --------------------------------- |
| `present`      | boolean | 必须 | 是否存在恢复记录                          |
| `method`       | string  | 可以 | 生成方法标识；本规范只定义保留值 `client-defined` |
| `coveredRange` | string  | 必须 | 恢复记录保护的范围                         |
| `length`       | uint64  | 必须 | 恢复记录字节数                           |

`coveredRange` 取值：

| 值                            | 含义           |
| ---------------------------- | ------------ |
| `central-directory`          | 仅保护中央目录      |
| `manifest`                   | 仅保护 manifest |
| `central-directory+manifest` | 保护两者（推荐）     |
| `full`                       | 保护整个文件尾部区域   |

**一致性约束**：`recovery.length` **必须**等于 Header/Footer 中 `recoveryLength` 的值；`recovery.present` 为 `true` 时 Header 的 `HAS_RECOVERY` 位**必须**置 1，反之亦然。三者不一致时客户端**应当**报告 `E_ZIP_STRUCTURE`。

***

## 5. 条目命名规范

| 路径模式                                    | 用途                  | 必需性              |
| --------------------------------------- | ------------------- | ---------------- |
| `manifest.json`                         | 索引文件                | 必须，且**必须**是第一个条目 |
| `manifests/r<revision>.json`            | 历史修订（可选保留）          | 可以               |
| `fonts/<sha256>.<ext>`                  | 字体文件                | 必须，至少一个          |
| `previews/<sha256>.preview.png`         | 单字体预览图              | 可以               |
| `previews/<sha256>-face<N>.preview.png` | 集合字体第 N 个 face 的预览图 | 可以               |
| `licenses/<sha256>.txt`                 | 许可证文本               | 可以               |
| `licenses/<sha256>.html`                | 许可证 HTML            | 可以               |
| `templates/<templateId>.json`           | 预览模板                | 可以               |
| `assets/cover.png`                      | 封面                  | 可以               |
| `assets/screenshots/<n>.png`            | 截图                  | 可以               |
| `assets/icons/<name>.png`               | 图标                  | 可以               |
| `.mfp-signature`                        | 签名 / 校验文件           | 可以               |

### 5.1 命名约束

- 字体条目名中的 `<sha256>` **必须**是该条目**内容**的 SHA-256，以 64 位小写十六进制表示
- 扩展名**必须**与字体实际格式一致（`ttf` / `otf` / `ttc` / `otc` / `woff` / `woff2`）
- 同一字体存在多种格式时，**必须**使用相同的哈希前缀、不同的扩展名
- 预览图、许可证的哈希前缀**必须**与对应字体条目的哈希前缀一致，便于关联

> **设计说明 8｜预览图命名统一**
> 预览图条目名统一为 `previews/<fontSha256>.preview.png`（单字体）与 `previews/<fontSha256>-face<N>.preview.png`（集合字体的第 N 个 face）。
> **理由**：若同时存在两套命名规则（例如 `<sha256>.preview.png` 与 `face-0-preview.png`），读取侧将无法可靠地从字体条目反推预览图条目名。

### 5.2 `.mfp-signature` 条目格式

可选的 JSON 条目，结构：

```json
{
  "algorithm": "sha256",
  "value": "sha256:...",
  "signedEntries": ["manifest.json", "fonts/a3f8c2d1....ttf"]
}
```

| 字段              | 类型        | 必需 | 说明                   |
| --------------- | --------- | -- | -------------------- |
| `algorithm`     | string    | 必须 | 签名/校验算法标识            |
| `value`         | string    | 必须 | 签名或校验值               |
| `signedEntries` | string\[] | 可以 | 被覆盖的条目名列表；缺省表示覆盖全部条目 |

本规范 v1 只要求该条目（若存在）为合法 JSON 且含 `algorithm` 与 `value` 字段；签名算法与验证逻辑由客户端定义。

> **设计说明 9｜`.mfp-signature`** **的内容格式**
> 规定该条目为 JSON，且至少含 `algorithm` 与 `value` 字段。
> **理由**：若完全不定义其格式，不同打包工具会写出互不兼容的签名条目，接收方无法通用地判断"这个包有没有签名"。

***

## 6. 版本兼容规则

### 6.1 formatVersion 语义

`formatVersion` 为**主版本号**，语义遵循：

- **主版本号变化**表示不兼容的格式变更（如布局改变、必需字段移除、约束语义改变）
- 客户端读取时，若文件的 `formatVersion` **大于**自身支持的最大版本，**必须**拒绝读取并提示用户升级客户端（错误码 `E_FORMAT_VERSION`）
- 若 `formatVersion` **小于**自身支持的最大版本，客户端**应当**尝试兼容读取，缺失字段用默认值填充

`mfpVersion` 为语义化版本字符串，仅作信息用途，**必须不**参与兼容性判定——兼容性判定**必须**以 `formatVersion` 为准。

### 6.2 向后兼容约束

**必需字段（不可删除、不可重命名、不可改变语义）**：

`mfpVersion`、`formatVersion`、`revision`、`fonts`、`checksum`

**可选字段（可以新增、可以缺失）**：

`parentRevision`、`operation`、`changes`、`templates`、`conflicts`、`storage`、`recovery`

新增可选字段**必须不**影响旧客户端读取。删除或重命名必需字段属于不兼容变更，**必须**提升 `formatVersion` 主版本号。

### 6.3 读取降级规则

| 场景                                      | 客户端行为                            |
| --------------------------------------- | -------------------------------- |
| manifest 缺失 `templates`                 | 使用内置默认模板                         |
| manifest 缺失 `storage`                   | 不执行自动紧凑化，由用户手动触发                 |
| manifest 缺失 `conflicts`                 | 跳过冲突提示                           |
| manifest 缺失 `recovery`                  | 视为无恢复记录                          |
| 字体条目缺失 `preview`                        | 读取时回退生成预览                        |
| 字体条目缺失 `metadata.variationAxes`         | 视为静态字体                           |
| 集合字体缺失 `faces`                          | 读取时解析字体文件动态获取                    |
| `parentRevision` 指向的历史条目不存在             | 视为到达链起点，**不报错**（见 4.2.3）         |
| `compactionThreshold` 超出 `[0.05, 0.50]` | 报告 `E_THRESHOLD_RANGE`，按最近的边界值处理 |

### 6.4 兼容性矩阵

以「客户端支持的最大 `formatVersion`」为 `N`：

| 文件 `formatVersion` | 客户端行为                          |
| ------------------ | ------------------------------ |
| `< N`              | 兼容读取，缺失字段按 6.3 降级              |
| `= N`              | 正常读取                           |
| `> N`              | **必须**拒绝，返回 `E_FORMAT_VERSION` |

**写入侧约束**：客户端写出的 `formatVersion` **必须**等于自身实现所遵循的规范版本，**必须不**写出高于自身支持能力的版本。

> **设计说明 12｜必须提供兼容性矩阵**
> 明确以「客户端支持的最大 `formatVersion`」为 `N`，并给出 `< N` / `= N` / `> N` 三种情形的行为。
> **理由**：没有这张表，实现者对"读到更高版本该拒绝还是尽力解析"会各自发挥，导致同一个文件在不同客户端上表现不一致。

***

## 7. 格式级约束汇总

以下为**必须**满足的格式级约束，逐条可验证：

1. **必须使用 ZIP64**：所有偏移与大小字段按 ZIP64 规范处理，ZIP64 EOCD Record 与 Locator 必须存在
2. **中央目录必须在文件末尾**：条目数据在前，CD 与 EOCD 结构收尾
3. **`manifest.json`** **必须是第一个条目**：便于流式读取优先获取索引
4. **MFP Header 必须位于文件起始 32 字节**：用于格式识别
5. **Footer Descriptor 必须紧跟 ZIP EOCD**：用于定位恢复记录
6. **`compactionThreshold`** **必须在** **`[0.05, 0.50]`**：防止极端配置
7. **字体文件以内容 SHA-256 命名**：保证内容寻址与去重
8. **模板只支持单字体渲染**：多字体模板视为无效
9. **恢复记录的算法由客户端定义**：格式只规定位置、长度与覆盖范围
10. **写入 Header 后必须修正 ZIP 偏移**：见 3.3.4，共 5 处字段
11. **`manifestOffset`** **必须指向 Local File Header 起始位置**：且每次修改容器后必须重写
12. **`headerCRC32`** **必须覆盖除自身外的全部 28 字节**：见 3.2.2
13. **所有多字节整数必须小端序**：见 3.5
14. **条目名必须 UTF-8 编码并置位 EFS 标志**：见 3.5

***

## 8. 格式识别流程

客户端打开文件时的标准识别路径：

```
1. 读取前 32 字节
   ├─ 校验 magic == "MFP\x01"
   │   ├─ 是 → 继续
   │   └─ 否 → 中止，返回 E_MAGIC
   ├─ 校验 formatVersion ≤ 客户端支持的最大版本
   │   ├─ 是 → 继续
   │   └─ 否 → 中止，返回 E_FORMAT_VERSION
   └─ 校验 headerCRC32
       ├─ 通过 → 继续
       └─ 失败 → 中止，返回 E_HEADER_CRC

2. 由 Header 的 manifestOffset 定位 manifest 条目的 Local File Header
   ├─ 校验该位置签名为 0x04034B50
   │   ├─ 是 → 继续
   │   └─ 否 → 中止，返回 E_ZIP_STRUCTURE
   └─ 流式读取 manifest 内容

3. 解析 manifest
   ├─ 结构校验（Schema）
   │   ├─ 通过 → 继续
   │   └─ 失败 → 中止，返回 E_MANIFEST_INVALID
   ├─ 检查 revision 链
   │   └─ 存在 parentRevision → 按 4.2.3 回溯重建索引（历史缺失视为终点）
   ├─ 校验 checksum（可选）
   │   ├─ 通过或跳过 → 继续
   │   └─ 失败 → 报告 E_CHECKSUM_MISMATCH，由用户决定是否继续
   └─ 校验 storage.compactionThreshold 区间
       └─ 越界 → 报告 E_THRESHOLD_RANGE，按边界值处理

4. 检查 ZIP EOCD 之后是否紧跟 "MFP\xFF"
   ├─ 是 → 解析 Footer Descriptor，读取恢复记录信息
   │   └─ 与 manifest.recovery 交叉校验，不一致则报告 E_ZIP_STRUCTURE
   └─ 否 → 若 HAS_FOOTER 置位则报告结构异常，否则跳过

5. 就绪，按需读取条目
```

### 8.1 读取侧完整性校验顺序

校验**必须**按以下顺序进行，任一步失败即中止（标注"可降级"的除外）：

| 顺序 | 校验项                  | 失败行为                                       |
| -- | -------------------- | ------------------------------------------ |
| 1  | magic                | 中止，`E_MAGIC`                               |
| 2  | `formatVersion`      | 中止，`E_FORMAT_VERSION`                      |
| 3  | `headerCRC32`        | 中止，`E_HEADER_CRC`                          |
| 4  | `manifestOffset` 处签名 | 中止，`E_ZIP_STRUCTURE`                       |
| 5  | manifest 存在性         | 中止，`E_MANIFEST_MISSING`                    |
| 6  | manifest Schema      | 中止，`E_MANIFEST_INVALID`                    |
| 7  | `checksum`           | **可降级**：报告 `E_CHECKSUM_MISMATCH`，由用户决定是否继续 |
| 8  | Footer 一致性           | **可降级**：报告 `E_ZIP_STRUCTURE`，无恢复记录时仍可读取    |
| 9  | 单个条目按需校验             | **可降级**：该条目不可用，其余条目不受影响                    |

**顺序设计理由**：先做廉价的头部校验（1–3），再做需要定位的校验（4），最后做需要解析的昂贵校验（5–6）。这样绝大多数损坏文件能在读入任何条目数据之前被拒绝。

> **设计说明 14｜必须固定完整性校验顺序**
> 校验必须按上表顺序执行，且明确哪些步骤失败后可降级。
> **理由**：顺序固定后，"一个损坏文件会报哪个错误"是可预期的，便于测试与用户支持；先廉价后昂贵的排序也让绝大多数损坏文件在读取任何条目数据之前就被拒绝。

***

## 9. 与客户端的接口约定

格式规范为客户端预留以下扩展点。表中"格式提供"是本规范定义的内容，"客户端实现"是客户端需自行决定的内容。

| 扩展点   | 格式提供                                      | 客户端实现                                     | 详见      |
| ----- | ----------------------------------------- | ----------------------------------------- | ------- |
| 格式识别  | magic、`formatVersion`、`headerCRC32`       | 读取与校验流程                                   | 第 8 节   |
| 增量更新  | `revision` 链、`changes` 字段、追加语义            | 截断 CD、追加条目、重写 CD/EOCD、重写 `manifestOffset` | 4.2     |
| 校验和   | `checksum` 结构                             | 哈希计算与验证时机                                 | 4.7     |
| 恢复记录  | Footer 位置、`recoveryLength`、`coveredRange` | 生成算法、修复逻辑                                 | 3.4、4.9 |
| 预览渲染  | 模板结构、变量列表                                 | 渲染引擎、字体加载、回退链                             | 4.5     |
| 元数据解析 | `metadata` 字段定义                           | 用字体引擎解析并填充                                | 4.4.5   |
| 冲突检测  | `conflicts` 记录结构                          | 检测算法、UI 呈现                                | 4.8     |
| 安装    | `installMode` 字段                          | 平台 API、注册表写入、提权                           | 4.4.4   |
| 回收    | `storage` 结构、阈值区间                         | 动态调整策略、紧凑化执行                              | 4.6     |

每个扩展点在客户端侧的具体落地方式（模块划分、接口签名、伪代码）见 [`IMPLEMENTATION.md`](./IMPLEMENTATION.md) 第 3、4 章。

***

## 10. 设计说明与实现约定

本章汇总对格式设计过程中形成的**关键决策与实现约定**。每条给出结论与理由。实现者**应当**逐条落实。

### 10.1 设计说明清单

| 编号 | 类型       | 议题                     | 结论                            |
| -- | -------- | ---------------------- | ----------------------------- |
| 1  | **关键约束** | Header 前置未修正 ZIP 偏移    | 见 3.3.4，必须修正 5 处字段            |
| 2  | 语义明确     | `manifestOffset` 指向不明确 | 见 3.2.3，指向 Local File Header  |
| 3  | 覆盖范围     | `headerCRC32` 覆盖范围过窄   | 见 3.2.2，覆盖 28 字节              |
| 4  | 覆盖范围     | `footerCRC32` 覆盖范围未定义  | 见 3.4，覆盖 20 字节                |
| 5  | 语义明确     | revision 历史存储位置含糊      | 见 4.2.3，`manifests/r<N>.json` |
| 6  | 约定       | 压缩方法未约定                | 见 10.2                        |
| 7  | 约定       | 数据描述符未约定               | 见 10.2                        |
| 8  | 统一       | 预览图命名规则冲突              | 见 5.1                         |
| 9  | 约定       | `.mfp-signature` 格式未定义 | 见 5.2                         |
| 10 | 补充       | 无错误码表                  | 见 10.3                        |
| 11 | 补充       | 无 JSON Schema          | 见 10.4、10.5                   |
| 12 | 补充       | 无兼容性矩阵                 | 见 6.4                         |
| 13 | 补充       | 无示例                    | 见 10.6                        |
| 14 | 补充       | 无完整性校验顺序               | 见 8.1                         |
| 15 | 修正       | flags 位 3 无信息量         | 见 3.2.1，改为 `HAS_HISTORY`      |

### 10.2 压缩方法与数据描述符约定

**压缩方法**：

| 条目类型                                       | 压缩方法                | 理由                                              |
| ------------------------------------------ | ------------------- | ----------------------------------------------- |
| `fonts/*`                                  | `store`（方法 0）       | 字体格式自身已压缩（尤其 WOFF/WOFF2），deflate 收益极低而 CPU 开销显著 |
| `manifest.json`、`templates/*`、`licenses/*` | `deflate`（方法 8）     | 文本内容压缩收益明显                                      |
| `previews/*`、`assets/*`                    | `store` 或 `deflate` | PNG 已压缩，`store` 更省 CPU                          |

> **设计说明 6｜压缩方法必须按条目类型约定**
> 字体条目用 `store`，文本条目用 `deflate`。
> **理由**：字体文件（尤其 WOFF/WOFF2）自身已是压缩格式，再走 deflate 几乎不减小体积却显著增加打包与读取的 CPU 开销。

**数据描述符**：

流式写入时条目大小在写入前未知，此时：

- Local File Header 中的 CRC32、压缩大小、未压缩大小字段**必须**填 `0xFFFFFFFF`（ZIP64 场景）
- 通用位标志**必须**置位 **bit 3**（data descriptor present）
- 数据之后**必须**写入数据描述符（签名 `0x08074B50`，随后是 CRC32 与 ZIP64 尺寸）
- 此时**必须**以**中央目录中记录的大小为准**，**必须不**信任 Local File Header 中的占位值

> **设计说明 7｜数据描述符的允许与读取优先级**
> 允许使用数据描述符，但规定"以中央目录中的大小为准"。
> **理由**：流式写入时 size 在写出前未知，描述符是唯一可行方案；但 Local File Header 中会留下 `0xFFFFFFFF` 占位值，若不明确读取优先级，实现者会读到错误的尺寸。

### 10.3 错误码表

客户端**应当**使用以下错误码标识失败原因，便于上层区分处理与提示：

| 错误码                   | 触发条件                                                            | 是否可恢复      |
| --------------------- | --------------------------------------------------------------- | ---------- |
| `E_MAGIC`             | 文件前 4 字节不是 `MFP\x01`                                            | 否          |
| `E_FORMAT_VERSION`    | `formatVersion` 高于客户端支持的最大版本                                    | 否（需升级客户端）  |
| `E_HEADER_CRC`        | `headerCRC32` 校验失败                                              | 否          |
| `E_ZIP_STRUCTURE`     | ZIP 结构异常（`manifestOffset` 处签名不符、EOCD 缺失、Footer 与 manifest 不一致等） | 否          |
| `E_MANIFEST_MISSING`  | 未找到 `manifest.json` 条目                                          | 否          |
| `E_MANIFEST_INVALID`  | manifest 不符合 Schema                                             | 否          |
| `E_ENTRY_MISSING`     | 请求的条目在容器中不存在                                                    | 是（跳过该条目）   |
| `E_CHECKSUM_MISMATCH` | `checksum` 校验失败                                                 | 是（用户可选择继续） |
| `E_TEMPLATE_INVALID`  | 模板不符合约束（如多字体定义、画布尺寸越界）                                          | 是（回退默认模板）  |
| `E_THRESHOLD_RANGE`   | `compactionThreshold` 超出 `[0.05, 0.50]`                         | 是（按边界值处理）  |
| `E_LOCKED`            | 容器被其他进程锁定，无法获取写入锁                                               | 是（重试）      |
| `E_COMPACTION_NEEDED` | 垃圾数据占比超阈值，需先紧凑化                                                 | 是（执行紧凑化）   |

> **设计说明 10｜必须提供错误码表**
> 上表是规范的一部分，客户端应当直接使用这些标识符。
> **理由**：统一错误码让上层能区分"文件不是 MFP"与"文件损坏"与"需要重试"，也让跨客户端的日志与用户支持可以互通。

### 10.4 manifest JSON Schema

以下 Schema 可直接落地为 `manifest.schema.json`：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://mfp.example/schema/manifest.schema.json",
  "title": "MFP manifest",
  "type": "object",
  "required": ["mfpVersion", "formatVersion", "revision", "fonts", "checksum"],
  "additionalProperties": true,
  "properties": {
    "mfpVersion": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "formatVersion": { "type": "integer", "minimum": 1 },
    "revision": { "type": "integer", "minimum": 1 },
    "parentRevision": { "type": ["integer", "null"], "minimum": 1 },
    "createdAt": { "type": "string", "format": "date-time" },
    "updatedAt": { "type": "string", "format": "date-time" },
    "operation": { "enum": ["create", "addFont", "removeFont", "updateMetadata", "compact"] },
    "changes": {
      "type": "object",
      "additionalProperties": true,
      "properties": {
        "added": { "type": "array", "items": { "type": "string" } },
        "removed": { "type": "array", "items": { "type": "string" } },
        "modified": { "type": "array", "items": { "type": "string" } },
        "compacted": { "type": "boolean" }
      }
    },
    "package": {
      "type": "object",
      "required": ["id", "name", "version"],
      "additionalProperties": true,
      "properties": {
        "id": { "type": "string", "minLength": 1 },
        "name": { "type": "string", "minLength": 1 },
        "description": { "type": "string" },
        "version": { "type": "string", "minLength": 1 },
        "cover": { "type": "string" },
        "screenshots": { "type": "array", "items": { "type": "string" } },
        "icon": { "type": "string" },
        "eula": { "type": "string" },
        "allowRedistribution": { "type": "boolean", "default": false },
        "tags": { "type": "array", "items": { "type": "string" } }
      }
    },
    "fonts": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["id", "hash", "file", "format", "isCollection", "faceCount", "installMode", "metadata"],
        "additionalProperties": true,
        "properties": {
          "id": { "type": "string", "minLength": 1 },
          "hash": { "type": "string", "pattern": "^sha256:[0-9a-f]{64}$" },
          "file": { "type": "string", "pattern": "^fonts/[0-9a-f]{64}\\.(ttf|otf|ttc|otc|woff|woff2)$" },
          "originalName": { "type": "string" },
          "format": { "enum": ["ttf", "otf", "ttc", "otc", "woff", "woff2"] },
          "isCollection": { "type": "boolean" },
          "faceCount": { "type": "integer", "minimum": 1 },
          "installMode": { "enum": ["singleFace", "allFaces"] },
          "faces": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["faceIndex", "postscriptName", "familyName", "subfamilyName"],
              "additionalProperties": true,
              "properties": {
                "faceIndex": { "type": "integer", "minimum": 0 },
                "postscriptName": { "type": "string" },
                "familyName": { "type": "string" },
                "subfamilyName": { "type": "string" },
                "version": { "type": "string" },
                "characterSet": {
                  "type": "object",
                  "additionalProperties": true,
                  "properties": {
                    "unicodeRanges": { "type": "array", "items": { "type": "string" } },
                    "glyphCount": { "type": "integer", "minimum": 0 }
                  }
                },
                "variationAxes": { "type": ["object", "null"] },
                "colorTables": { "type": "array", "items": { "type": "string" } }
              }
            }
          },
          "metadata": {
            "type": "object",
            "required": ["familyName", "subfamilyName", "postscriptName", "fullName"],
            "additionalProperties": true,
            "properties": {
              "familyName": { "type": "string" },
              "subfamilyName": { "type": "string" },
              "postscriptName": { "type": "string" },
              "fullName": { "type": "string" },
              "version": { "type": "string" },
              "copyright": { "type": "string" },
              "designer": { "type": "string" },
              "designerURL": { "type": "string" },
              "license": { "type": "string" },
              "licenseURL": { "type": "string" },
              "characterSet": {
                "type": "object",
                "additionalProperties": true,
                "properties": {
                  "unicodeRanges": { "type": "array", "items": { "type": "string" } },
                  "glyphCount": { "type": "integer", "minimum": 0 }
                }
              },
              "variationAxes": { "type": ["object", "null"] },
              "colorTables": { "type": "array", "items": { "type": "string" } },
              "availableFormats": { "type": "array", "items": { "type": "string" } },
              "woff2File": { "type": "string" }
            }
          },
          "preview": {
            "type": "object",
            "additionalProperties": true,
            "properties": {
              "image": { "type": "string" },
              "templateId": { "type": "string" },
              "generatedText": { "type": "string" },
              "source": { "enum": ["user", "generated", "embedded", "fallback"] }
            }
          },
          "license": {
            "type": "object",
            "additionalProperties": true,
            "properties": {
              "type": { "type": "string" },
              "file": { "type": "string" },
              "allowRedistribution": { "type": "boolean" },
              "allowCommercial": { "type": "boolean" },
              "allowModification": { "type": "boolean" }
            }
          }
        }
      }
    },
    "templates": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "name", "file"],
        "additionalProperties": true,
        "properties": {
          "id": { "type": "string", "minLength": 1 },
          "name": { "type": "string", "minLength": 1 },
          "isDefault": { "type": "boolean" },
          "file": { "type": "string", "pattern": "^templates/.+\\.json$" }
        }
      }
    },
    "conflicts": {
      "type": "object",
      "additionalProperties": true,
      "properties": {
        "duplicates": { "$ref": "#/$defs/conflictList" },
        "versionConflicts": { "$ref": "#/$defs/conflictList" },
        "nameConflicts": { "$ref": "#/$defs/conflictList" },
        "formatDuplicates": { "$ref": "#/$defs/conflictList" }
      }
    },
    "storage": {
      "type": "object",
      "additionalProperties": true,
      "properties": {
        "garbageBytes": { "type": "integer", "minimum": 0 },
        "garbageRatio": { "type": "number", "minimum": 0, "maximum": 1 },
        "compactionThreshold": { "type": "number", "minimum": 0.05, "maximum": 0.5 },
        "lastCompactionRevision": { "type": "integer", "minimum": 0 },
        "entryCount": { "type": "integer", "minimum": 0 },
        "liveEntryCount": { "type": "integer", "minimum": 0 }
      }
    },
    "checksum": {
      "type": "object",
      "required": ["algorithm", "manifestHash", "entries"],
      "additionalProperties": true,
      "properties": {
        "algorithm": { "enum": ["sha256"] },
        "manifestHash": { "type": "string" },
        "entries": {
          "type": "object",
          "additionalProperties": { "type": "string" }
        }
      }
    },
    "recovery": {
      "type": "object",
      "required": ["present", "coveredRange", "length"],
      "additionalProperties": true,
      "properties": {
        "present": { "type": "boolean" },
        "method": { "type": "string" },
        "coveredRange": {
          "enum": ["central-directory", "manifest", "central-directory+manifest", "full"]
        },
        "length": { "type": "integer", "minimum": 0 }
      }
    }
  },
  "$defs": {
    "conflictList": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["fontId", "type"],
        "additionalProperties": true,
        "properties": {
          "fontId": { "type": "string" },
          "duplicateOf": { "type": "string" },
          "conflictsWith": { "type": "string" },
          "type": { "enum": ["exact", "version", "name", "format"] }
        }
      }
    }
  }
}
```

> **设计说明 11｜必须提供 JSON Schema**
> 规范内嵌完整的 manifest 与模板 JSON Schema（本节与 10.5 节），可直接落地为 `schema/*.schema.json`。
> **理由**：Schema 让"结构合法"成为可自动校验的判据，避免各实现者对字段类型与取值范围的解读分歧；也让打包侧能在写出前自检。

### 10.5 模板 JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://mfp.example/schema/template.schema.json",
  "title": "MFP preview template",
  "type": "object",
  "required": ["templateId", "canvas", "textLayers"],
  "additionalProperties": true,
  "properties": {
    "templateId": { "type": "string", "minLength": 1 },
    "name": { "type": "string" },
    "canvas": {
      "type": "object",
      "required": ["width", "height"],
      "additionalProperties": true,
      "properties": {
        "width": { "type": "integer", "minimum": 1, "maximum": 4096 },
        "height": { "type": "integer", "minimum": 1, "maximum": 4096 },
        "background": { "type": "string", "pattern": "^#[0-9A-Fa-f]{6}([0-9A-Fa-f]{2})?$" }
      }
    },
    "textLayers": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["text", "fontSize", "x", "y"],
        "additionalProperties": true,
        "properties": {
          "id": { "type": "string" },
          "text": { "type": "string" },
          "fontSize": { "type": "number", "exclusiveMinimum": 0 },
          "color": { "type": "string", "pattern": "^#[0-9A-Fa-f]{6}([0-9A-Fa-f]{2})?$" },
          "x": { "type": "number" },
          "y": { "type": "number" },
          "align": { "enum": ["left", "center", "right"] },
          "baseline": { "enum": ["top", "middle", "bottom", "alphabetic"] }
        }
      }
    },
    "decorations": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["type", "x", "y", "width", "height"],
        "additionalProperties": true,
        "properties": {
          "type": { "enum": ["rect"] },
          "x": { "type": "number" },
          "y": { "type": "number" },
          "width": { "type": "number", "minimum": 0 },
          "height": { "type": "number", "minimum": 0 },
          "fill": { "type": "string" }
        }
      }
    },
    "fallback": {
      "type": "object",
      "additionalProperties": true,
      "properties": {
        "text": { "type": "string" },
        "fontSize": { "type": "number", "exclusiveMinimum": 0 },
        "color": { "type": "string" }
      }
    }
  }
}
```

### 10.6 示例

#### 10.6.1 最小 manifest

一个满足全部必需字段的单字体包，无模板、无冲突记录、无恢复记录：

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
      "hash": "sha256:2b1a7c9e4d3f8a6b5c2e1d0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b",
      "file": "fonts/2b1a7c9e4d3f8a6b5c2e1d0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b.ttf",
      "originalName": "MyFont-Regular.ttf",
      "format": "ttf",
      "isCollection": false,
      "faceCount": 1,
      "installMode": "singleFace",
      "metadata": {
        "familyName": "My Font",
        "subfamilyName": "Regular",
        "postscriptName": "MyFont-Regular",
        "fullName": "My Font Regular",
        "version": "1.000",
        "characterSet": {
          "unicodeRanges": ["U+0020-U+007E"],
          "glyphCount": 234
        },
        "variationAxes": null,
        "colorTables": []
      }
    }
  ],
  "checksum": {
    "algorithm": "sha256",
    "manifestHash": "sha256:0000000000000000000000000000000000000000000000000000000000000000",
    "entries": {
      "fonts/2b1a7c9e4d3f8a6b5c2e1d0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b.ttf": "sha256:2b1a7c9e4d3f8a6b5c2e1d0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b"
    }
  }
}
```

对应的最小文件结构：

```
[MFP Header 32B，HAS_RECOVERY=0, HAS_FOOTER=0, HAS_HISTORY=0]
[Local File Header + manifest.json（deflate）]
[Local File Header + fonts/2b1a7c9e...ttf（store）]
[Central Directory]
[ZIP64 EOCD Record]
[ZIP64 EOCD Locator]
[EOCD Record]
```

#### 10.6.2 完整 manifest

覆盖集合字体、模板、冲突、存储与恢复记录的完整示例：

```json
{
  "mfpVersion": "1.0.0",
  "formatVersion": 1,
  "revision": 42,
  "parentRevision": 41,
  "createdAt": "2026-09-20T12:00:00Z",
  "updatedAt": "2026-09-20T15:30:00Z",
  "operation": "addFont",
  "changes": {
    "added": ["font-005"],
    "removed": ["font-002"],
    "modified": ["fonts/font-001", "templates/default-v1"],
    "compacted": false
  },
  "package": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "My Font Pack",
    "description": "A collection of display fonts",
    "version": "2.1.0",
    "cover": "assets/cover.png",
    "screenshots": ["assets/screenshots/01.png"],
    "icon": "assets/icons/icon.png",
    "eula": "licenses/eula.txt",
    "allowRedistribution": true,
    "tags": ["display", "serif"]
  },
  "fonts": [
    {
      "id": "font-001",
      "hash": "sha256:a3f8c2d1e5b7a9f4c6d8e0b2a4c6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2",
      "file": "fonts/a3f8c2d1e5b7a9f4c6d8e0b2a4c6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2.ttc",
      "originalName": "MyFontCollection.ttc",
      "format": "ttc",
      "isCollection": true,
      "faceCount": 4,
      "installMode": "allFaces",
      "faces": [
        {
          "faceIndex": 0,
          "postscriptName": "MyFont-Regular",
          "familyName": "My Font",
          "subfamilyName": "Regular",
          "version": "1.000",
          "characterSet": { "unicodeRanges": ["U+0020-U+007E"], "glyphCount": 567 },
          "variationAxes": null,
          "colorTables": []
        },
        {
          "faceIndex": 1,
          "postscriptName": "MyFont-Bold",
          "familyName": "My Font",
          "subfamilyName": "Bold",
          "version": "1.000",
          "characterSet": { "unicodeRanges": ["U+0020-U+007E"], "glyphCount": 571 },
          "variationAxes": null,
          "colorTables": []
        }
      ],
      "metadata": {
        "familyName": "My Font",
        "subfamilyName": "Regular",
        "postscriptName": "MyFont-Regular",
        "fullName": "My Font Regular",
        "version": "1.000",
        "copyright": "Copyright 2026",
        "designer": "Jane Doe",
        "designerURL": "https://example.com",
        "license": "OFL-1.1",
        "licenseURL": "https://scripts.sil.org/OFL",
        "characterSet": {
          "unicodeRanges": ["U+0020-U+007E", "U+00A0-U+00FF"],
          "glyphCount": 1234
        },
        "variationAxes": {
          "wght": { "name": "Weight", "min": 100, "default": 400, "max": 900 }
        },
        "colorTables": ["COLR", "CPAL"],
        "availableFormats": ["ttf", "woff2"],
        "woff2File": "fonts/a3f8c2d1e5b7a9f4c6d8e0b2a4c6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2.woff2"
      },
      "preview": {
        "image": "previews/a3f8c2d1e5b7a9f4c6d8e0b2a4c6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2.preview.png",
        "templateId": "default-v1",
        "generatedText": "AaBbCc 你好世界 0123",
        "source": "generated"
      },
      "license": {
        "type": "OFL-1.1",
        "file": "licenses/a3f8c2d1e5b7a9f4c6d8e0b2a4c6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2.license.txt",
        "allowRedistribution": true,
        "allowCommercial": true,
        "allowModification": true
      }
    }
  ],
  "templates": [
    {
      "id": "default-v1",
      "name": "默认预览模板",
      "isDefault": true,
      "file": "templates/default-v1.json"
    }
  ],
  "conflicts": {
    "duplicates": [
      { "fontId": "font-002", "duplicateOf": "font-001", "type": "exact" }
    ],
    "versionConflicts": [
      { "fontId": "font-003", "conflictsWith": "font-001", "type": "version" }
    ],
    "nameConflicts": [
      { "fontId": "font-004", "conflictsWith": "font-001", "type": "name" }
    ],
    "formatDuplicates": [
      { "fontId": "font-005", "duplicateOf": "font-001", "type": "format" }
    ]
  },
  "storage": {
    "garbageBytes": 52428800,
    "garbageRatio": 0.12,
    "compactionThreshold": 0.2,
    "lastCompactionRevision": 30,
    "entryCount": 47,
    "liveEntryCount": 42
  },
  "checksum": {
    "algorithm": "sha256",
    "manifestHash": "sha256:0000000000000000000000000000000000000000000000000000000000000000",
    "entries": {
      "fonts/a3f8c2d1e5b7a9f4c6d8e0b2a4c6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2.ttc": "sha256:a3f8c2d1e5b7a9f4c6d8e0b2a4c6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2",
      "previews/a3f8c2d1e5b7a9f4c6d8e0b2a4c6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2.preview.png": "sha256:2222222222222222222222222222222222222222222222222222222222222222",
      "templates/default-v1.json": "sha256:3333333333333333333333333333333333333333333333333333333333333333"
    }
  },
  "recovery": {
    "present": true,
    "method": "client-defined",
    "coveredRange": "central-directory+manifest",
    "length": 1048576
  }
}
```

对应的 Header / Footer 取值：

| 位置                      | 取值                                                                    |
| ----------------------- | --------------------------------------------------------------------- |
| Header `flags`          | `HAS_RECOVERY=1`、`HAS_FOOTER=1`、`HAS_HISTORY=1` → `0b1110` = `0x000E` |
| Header `manifestOffset` | 32（manifest 是第一个条目，紧邻 Header 之后）                                      |
| Footer `recoveryLength` | `1048576`（与 `recovery.length` 一致）                                     |

> **设计说明 13｜必须提供最小与完整示例**
> 本节给出一个最小可用 manifest（单字体、无模板、无恢复记录）与一个覆盖全部字段的完整 manifest。
> **理由**：示例是规范落地的最短路径——实现者可以先照着示例产出可解析的容器，再逐步补齐可选字段；同时也为测试提供基准数据。

***

## 11. 附录

### 11.1 推荐仓库形态

本规范与实现指南当前位于 `mfp-spec/` 目录。若要把协议独立开源，建议补齐为以下形态：

```
mfp-spec/
├── SPEC.md                    # 完整规范（本文件）
├── IMPLEMENTATION.md          # 客户端实现指南
├── CHANGELOG.md               # 各 formatVersion 的变更记录
├── schema/
│   ├── manifest.schema.json   # 取自 10.4
│   └── template.schema.json   # 取自 10.5
├── examples/
│   ├── minimal.json           # 取自 10.6.1
│   ├── full.json              # 取自 10.6.2
│   └── layout.md              # 最小文件结构的字节级示意
└── LICENSE                    # MIT
```

拆分后，JSON Schema 可独立被客户端在读取时用于结构校验、在打包时用于生成校验，规范本身不依赖任何具体实现库。

### 11.2 与实现文档的对应关系

| 本规范章节          | [`IMPLEMENTATION.md`](./IMPLEMENTATION.md) 对应内容 |
| -------------- | ----------------------------------------------- |
| 第 3 节 文件物理布局   | `mfp-format.js`（Header/Footer 编解码、偏移修正）         |
| 第 4 节 manifest | `mfp-manifest.js`（读写、Schema 校验、revision 链重建）    |
| 3.3 ZIP 布局     | `mfp-archive.js`（ZIP64 读写适配层）                   |
| 4.2 增量更新       | `mfp-append.js`（追加与 CD 重写）、`mfp-lock.js`（并发锁）   |
| 4.6 回收         | `mfp-compact.js`（动态阈值与紧凑化）                      |
| 4.8 冲突         | `mfp-conflict.js`（去重与冲突检测）                      |
| 4.5 预览模板       | `mfp-preview.js`（模板渲染与回退链）                      |
| 4.4.4 安装       | `mfp-install.js`（提取并调用宿主安装接口）                   |
| 第 9 节 扩展点      | 实现文档第 5 章「宿主环境接口契约」                             |

### 11.3 版本历史

| 版本    | formatVersion | 变更                       |
| ----- | ------------- | ------------------------ |
| 1.0.0 | 1             | 首版。含第 10.1 节列出的 15 项设计说明 |

