# .mfp 客户端实现指南

> 实现 `.mfp`（多媒体字体包）格式的客户端
> 配套协议规范见 [`SPEC.md`](./SPEC.md)

***

## 1. 文档说明

### 1.1 范围

本文档描述如何实现 `.mfp` 格式的**打包、读取、安装、增量更新**能力。内容包括：

- 参考架构分层与宿主环境能力要求
- 模块划分与文件清单
- 各模块的接口签名
- 宿主环境接口契约
- 核心流程的函数签名与伪代码
- 依赖建议、风险边界、分阶段落地与验收清单

本文档面向任意技术栈的实现者。伪代码采用 JS 风格书写、但函数名与参数名保持中立，其他语言可按同样的结构直接映射。

### 1.2 前提

> **本文档为设计文档，不包含任何可直接运行的完整实现。** 文中出现的所有模块、函数签名、接口均为**建议**，实现者可按自身架构调整。

伪代码中的错误码、字段名、条目名与 [`SPEC.md`](./SPEC.md) 保持一致，可直接对照阅读。

### 1.3 术语

与 [`SPEC.md`](./SPEC.md) 第 1.3 节一致。本文档额外使用：

| 术语              | 含义                                           |
| --------------- | -------------------------------------------- |
| **会话（Session）** | 一次「打开某个 .mfp 容器」的生命周期，由 `sessionId` 标识       |
| **打包侧**         | 生成 .mfp 的代码路径（可以是客户端内的「导出为字体包」功能，也可以是独立 CLI） |
| **客户端侧**        | 读取 .mfp、展示与安装的代码路径                           |
| **宿主（Host）**    | 承载核心库运行的环境：提供文件读写、字体安装、位图渲染等平台能力             |

***

## 2. 参考架构分层

### 2.1 分层模型

核心库自下而上分为 9 层。**上层只能依赖下层**，不得跨层反向调用。

| 层   | 职责                                 | 对应模块                             |
| --- | ---------------------------------- | -------------------------------- |
| 字节层 | Header/Footer 编解码、CRC32、偏移修正       | `mfp-format.js`                  |
| 归档层 | ZIP64 读写、条目命名、压缩方法选择               | `mfp-archive.js`                 |
| 索引层 | manifest 读写、Schema 校验、revision 链重建 | `mfp-manifest.js`                |
| 并发层 | 跨进程文件锁、动态 stale、续期                 | `mfp-lock.js`                    |
| 变更层 | 增量追加、紧凑化、垃圾回收                      | `mfp-append.js`、`mfp-compact.js` |
| 语义层 | 去重、冲突检测                            | `mfp-conflict.js`                |
| 表现层 | 预览模板渲染与回退链                         | `mfp-preview.js`                 |
| 落地层 | 提取到临时文件、调用宿主安装接口                   | `mfp-install.js`                 |
| 会话层 | 打开包的生命周期与授权                        | `mfp-session.js`                 |

### 2.2 宿主环境能力要求

实现者需要宿主提供以下能力。标注"可依赖第三方库"的项无需自行实现。

| 能力       | 最低要求                                                          | 说明                                                |
| -------- | ------------------------------------------------------------- | ------------------------------------------------- |
| 文件随机读写   | `open` / `read(offset, len)` / `write` / `truncate` / `close` | 增量追加与偏移修正的前提                                      |
| 64 位偏移运算 | 支持 > 4 GB 的偏移                                                 | 需 64 位整数类型（`BigInt` / `int64` / `u64`），**必须不**用浮点 |
| ZIP64 读写 | 能创建/读取 ZIP64 归档                                               | 可依赖第三方库                                           |
| SHA-256  | 流式哈希                                                          | 内容寻址命名                                            |
| CRC32    | 标准 ZIP CRC-32（多项式 `0xEDB88320`，反射，初值 `0xFFFFFFFF`，输出取反）       | 头尾校验                                              |
| 文件锁      | 跨进程互斥 + 可续期                                                   | 无则需自研（见 2.3）                                      |
| 字体元数据解析  | 读取 family / subfamily / PostScript 名 / 字符集 / 可变轴 / 颜色表        | 可依赖字体引擎库                                          |
| 字体安装     | 平台字体注册                                                        | 平台相关，无跨平台统一方案                                     |
| 位图渲染     | 2D Canvas 或等价能力                                               | 预览图生成                                             |

### 2.3 需自研的范围

以下能力**没有现成的第三方库可直接覆盖**，必须自行实现：

| 能力                         | 为何需自研                                                          |
| -------------------------- | -------------------------------------------------------------- |
| 偏移修正（5 处字段 `+= 32`）        | 通用 ZIP 库不支持「在 ZIP 前插入字节后修补中央目录」，见 [`SPEC.md`](./SPEC.md) 3.3.4 |
| 增量追加（截断 CD → 追加条目 → 重建 CD） | 通用 ZIP 库只支持一次性写出，不支持在已有归档上追加                                   |
| 动态锁 stale 与续期              | 固定 stale 值在大容器上会误判过期，导致并发写入损坏容器                                |
| 紧凑化阈值动态调整                  | 需结合容器大小、更新频率、磁盘余量综合判断                                          |
| revision 链重建与降级            | 格式特有语义，见 [`SPEC.md`](./SPEC.md) 4.2.3                          |

***

## 3. 模块划分

### 3.1 核心模块清单

| 文件                | 职责                                                        | 依赖的规范章节           |
| ----------------- | --------------------------------------------------------- | ----------------- |
| `index.js`        | 对外统一入口，re-export 各模块函数                                    | —                 |
| `mfp-format.js`   | Header / Footer 编解码、magic 常量、CRC32、**偏移修正（±32）**          | 3.2、3.4、3.5、3.3.4 |
| `mfp-manifest.js` | manifest 读写、Schema 校验、revision 链重建                        | 4.1–4.9、10.4      |
| `mfp-archive.js`  | ZIP64 读写适配层、条目命名与压缩方法策略                                   | 3.3、5.1、10.2      |
| `mfp-lock.js`     | 跨进程文件锁封装，**动态 stale**                                     | 4.6、10.3          |
| `mfp-append.js`   | 增量追加：定位 CD → 截断 → 追加条目 → 重写 CD/EOCD → 重写 `manifestOffset` | 4.2               |
| `mfp-compact.js`  | 垃圾回收与完全重写，**动态阈值**判定                                      | 4.6               |
| `mfp-conflict.js` | 去重与冲突检测                                                   | 4.8               |
| `mfp-preview.js`  | 预览模板渲染与回退链                                                | 4.5               |
| `mfp-install.js`  | 从包内提取字体到临时目录 → 调用宿主安装接口                                   | 4.4.4             |
| `mfp-session.js`  | 会话级授权与已打开包状态                                              | 8.1               |

### 3.2 模块依赖关系

```
                     mfp-session.js
                          │
      ┌───────────┬───────┼────────┬────────────┐
      ▼           ▼       ▼        ▼            ▼
 mfp-archive  mfp-lock mfp-manifest mfp-install  │
      │           │       │        │            │
      └───────────┴───────┘        │            │
                │                  │            │
                ▼                  ▼            ▼
          mfp-format.js ◄── mfp-append.js   宿主安装接口
                ▲              │
                │              ▼
          mfp-compact.js  mfp-conflict.js
                │              │
                └──────┬───────┘
                       ▼
                 mfp-preview.js
                       │
                       ▼
                 字体引擎（第三方）
```

**约束**：

- `mfp-format.js` 是唯一直接操作 Header/Footer 字节的模块，其余模块**必须不**自行拼接头部
- `mfp-install.js` 是唯一调用宿主安装接口的模块
- `mfp-archive.js` 是唯一直接依赖 ZIP 库的模块，便于将来替换实现
- 所有涉及写容器的路径**必须**经 `mfp-lock.js` 加锁

***

## 4. 接口签名

以下签名中 `→` 后为返回值；标注 `async` 的为异步接口。

### 4.1 `mfp-format.js`

```js
// ── 常量 ──
export const MFP_HEADER_BYTES = 32
export const MFP_HEADER_OFFSET_DELTA = 32   // Header 前置导致的 ZIP 偏移位移量

// ── Header ──
// 编码为 32 字节 Buffer；会自动计算 headerCRC32
export function encodeHeader({ formatVersion, flags, createdAt, manifestOffset })  // → Buffer(32)

// 解析 Header；校验 magic 与 headerCRC32，失败抛出带 errorCode 的错误
export function decodeHeader(headerBuffer)  // → { formatVersion, flags, createdAt, manifestOffset }

// flags 位操作
export function readFlags(flags)   // → { hasRecovery, hasFooter, hasHistory }
export function writeFlags(flagsObject)  // → uint16

// ── Footer ──
// 编码 Footer Descriptor（24 字节）+ 前置的 Recovery Record
export function encodeFooter({ recoveryBuffer })  // → Buffer（recoveryLength + 24）

// 从文件尾部解析 Footer；无 Footer 时返回 null
export async function decodeFooter(fileHandle, zipEocdOffset)  // → { recoveryLength, recoveryOffset } | null

// ── 偏移修正（规范 3.3.4，关键） ──
// 在写完 ZIP 并把 Header 前置之后调用，修补 5 处字段
export async function patchZipOffsetsAfterHeaderPrepend(filePath)  // → { patchedEntries: number }

// ── 工具 ──
export function crc32(buffer)  // → uint32
export function readUInt64LE(buffer, offset)  // → 64 位整数
export function writeUInt64LE(buffer, value, offset)  // → void
```

**`patchZipOffsetsAfterHeaderPrepend`** **的职责**（规范 3.3.4 的 5 处修正）：

```
1. 从文件尾部向前搜索 EOCD 签名 0x06054B50
2. 由 EOCD 定位 ZIP64 EOCD Locator（0x07064B50）与 ZIP64 EOCD（0x06064B50）
3. 定位中央目录起始位置
4. 遍历中央目录条目：
   a. 读 relative offset of local header（4B）→ += 32
   b. 若存在 ZIP64 Extended Information Extra Field（0x0001）且其中含 local header offset
      → += 32
5. EOCD 的 offset of start of central directory → += 32
6. ZIP64 EOCD 的 offset of start of central directory → += 32
7. ZIP64 EOCD Locator 的 relative offset of ZIP64 EOCD → += 32
8. 若 offset 超过 0xFFFFFFFF，必须确保 ZIP64 扩展字段存在
```

### 4.2 `mfp-manifest.js`

```js
// 从容器读取并解析 manifest（内部做 Schema 校验）
export async function readManifest(archiveHandle, options)  // → { manifest, manifestBuffer }
//   options: { verifyChecksum?: boolean, schemaPath? }

// 校验 manifest 结构
export function validateManifest(manifest)  // → { valid: boolean, errors: string[] }

// 沿 parentRevision 回溯重建索引；历史缺失视为链终点（规范 4.2.3）
export async function resolveRevisionChain(archiveHandle, manifest, options)
//   → { fonts: FontEntry[], templates: TemplateEntry[], resolvedRevisions: number[] }

// 计算 manifestHash（manifestHash 字段置空后计算，规范 4.7）
export function computeManifestHash(manifest)  // → "sha256:..."

// 组装 manifest 对象（打包侧用）
export function buildManifest(input)  // → manifest 对象
//   input: { package, fonts, templates?, conflicts?, storage?, recovery?, revision, parentRevision, operation, changes }
```

### 4.3 `mfp-archive.js`

```js
// ── 写 ──
// 创建 ZIP64 写入器；返回的对象支持流式追加条目
export function createArchiveWriter(outputPath, options)
//   options: { compressionPerEntry? }
//   → { appendEntry(name, source, options), finalize() }

// 条目级选项：压缩方法
//   options: { method: 'store' | 'deflate' }

// ── 读 ──
// 打开容器（只读中央目录，不加载全部数据）
export async function openArchive(filePath, options)  // → ArchiveHandle
//   ArchiveHandle: {
//     entries: Map<name, EntryInfo>,       // EntryInfo: { name, size, compressedSize, method }
//     readEntry(name, options): Readable,  // options: { start?, end? }
//     readEntryBuffer(name, options): Promise<Buffer>,
//     close(): Promise<void>
//   }

// ── 命名 ──
export function fontEntryName(sha256, format)             // → "fonts/<sha>.ttf"
export function previewEntryName(sha256, faceIndex)       // → "previews/<sha>.preview.png" | "previews/<sha>-face<N>.preview.png"
export function licenseEntryName(sha256, extension)       // → "licenses/<sha>.txt"
export function templateEntryName(templateId)             // → "templates/<id>.json"
export function historyEntryName(revision)                // → "manifests/r<revision>.json"

// ── 压缩方法策略（规范 10.2） ──
export function compressionFor(entryName)  // → 'store' | 'deflate'
```

### 4.4 `mfp-lock.js`

```js
// 获取容器写入锁；动态 stale
export async function acquireLock(filePath, options)
//   options: { estimatedDurationMs?, retries?, minTimeout?, maxTimeout? }
//   → release(): Promise<void>

// 按预估写入耗时计算 stale，并夹在安全区间内
export function computeStale(estimatedDurationMs)  // → ms

// 写入过程中续期（长耗时操作必须定期调用）
export async function refreshLock(release, filePath)  // → void
```

**`computeStale`** **的语义**：

```
基础值 = 30000 ms
stale = clamp(基础值 + 预估耗时 × 3, 30000, 600000)
```

**理由**：对超过 4 GB 容器的紧凑化，写入可能持续数分钟，固定 30 秒的 `stale` 会让其他进程误判锁已过期并抢占，导致容器损坏。同时**必须**在写入过程中定期 `refreshLock` 续期。

### 4.5 `mfp-append.js`

```js
// 增量追加条目（规范 4.2）
export async function appendEntries(filePath, entries, options)
//   entries: Array<{ name, source: Buffer | string, method? }>
//   options: { newManifest, estimatedDurationMs? }
//   → { revision, newManifestOffset, garbageBytes }

// 仅更新 manifest（不新增字体，如删除字体、改元数据）
export async function replaceManifest(filePath, newManifest, options)
//   → { revision, newManifestOffset }

// 读取当前容器的存储状态（用于计算 garbageRatio）
export async function readStorageState(filePath)  // → { totalBytes, garbageBytes, garbageRatio, entryCount, liveEntryCount }
```

**`appendEntries`** **的步骤**（对应规范 4.2 的追加语义）：

```
1. 获取写入锁（动态 stale）
2. 打开文件句柄（读写模式）
3. 读 EOCD，定位中央目录起始偏移 cdOffset（64 位整数）
4. 解析现有中央目录条目 → existingEntries
5. truncate(filePath, cdOffset)                      // 截掉 CD 与 EOCD
6. 逐个追加新条目：
   a. 写 Local File Header（签名 0x04034B50）
   b. 写数据（按 compressionFor 决定压缩方法）
   c. 若使用数据描述符，写 0x08074B50 + CRC32 + ZIP64 尺寸
7. 追加新的 manifest 条目（revision = 旧 revision + 1）
8. 重建中央目录 = existingEntries + 新条目（含新 manifest）
9. 写 Central Directory
10. 写 ZIP64 EOCD Record、ZIP64 EOCD Locator、EOCD Record
11. 重算并重写 Header 的 manifestOffset 与 headerCRC32   ← 规范 3.2.3 强制要求
12. 若启用历史保留，追加 manifests/r<旧 revision>.json
13. 关闭句柄
14. 释放锁（必须在 finally 中）
```

**安全边界**：

- `cdOffset` **必须**用 64 位整数处理；若宿主 API 只接受普通整数（如 JS 的 `number`），转换前**必须**校验 `cdOffset <= Number.MAX_SAFE_INTEGER`，否则抛出明确错误而非静默截断
- 第 5 步之后、第 11 步之前若进程崩溃，容器将处于不一致状态。因此**必须**在写 Header 前先完成 ZIP 结构，让 Header 的 CRC 成为「容器完整」的最后一道标记

### 4.6 `mfp-compact.js`

```js
// 计算当前垃圾比例
export async function computeGarbageRatio(filePath)  // → { ratio, garbageBytes, totalBytes }

// 动态阈值（规范 4.6：区间 [0.05, 0.50] 内的动态调整）
export function computeCompactionThreshold(context)
//   context: { totalBytes, updateFrequencyPerWeek, freeDiskBytes, lastCompactionMs }
//   → number in [0.05, 0.50]

// 判断是否需要紧凑化
export async function shouldCompact(filePath, context)  // → { needed, ratio, threshold }

// 执行紧凑化：读出全部有效条目，写新文件，原子替换
export async function compact(filePath, options)
//   options: { onProgress? }
//   → { newRevision, beforeBytes, afterBytes }
```

**`computeCompactionThreshold`** **的建议策略**：

```
起点 0.20
总大小 > 2GB            → +0.10（大文件容忍更高垃圾比例）
每周更新次数 > 10       → -0.05（高频更新宜更早回收）
剩余磁盘 < 容器大小×3   → -0.08（空间紧张宜更早回收）
上次紧凑化耗时 > 60s    → +0.05（避免频繁付出高代价）
最后 clamp 到 [0.05, 0.50]
```

**紧凑化的原子性**：**必须**写临时文件 → `fsync` → `rename` 覆盖原文件，**必须不**原地重写。原地重写期间崩溃会丢失全部数据。

### 4.7 `mfp-conflict.js`

```js
// 对一批新字体做去重与冲突检测（规范 4.8）
export function detectConflicts(newFonts, existingFonts)
//   newFonts / existingFonts: Array<{ id, hash, metadata, faces? }>
//   → { duplicates: [], versionConflicts: [], nameConflicts: [], formatDuplicates: [] }

// 生成去重后的字体条目（跳过 exact 重复）
export function deduplicateFonts(fonts)  // → { kept: [], skipped: [] }

// 与宿主已安装字体交叉检查
export async function checkInstalledConflicts(fonts, installChecker)
//   installChecker: (font) => Promise<{ installed: boolean, matches: [] }>
//   → Map<fontId, { installed, matches }>
```

**检测规则**（规范 4.8 的四类）：

| 判定     | 条件                                                    | 归入                                  |
| ------ | ----------------------------------------------------- | ----------------------------------- |
| 完全重复   | `hash` 相同                                             | `duplicates` (type `exact`)         |
| 版本冲突   | `metadata.postscriptName` 相同、`metadata.version` 不同    | `versionConflicts` (type `version`) |
| 同名不同字体 | `familyName` + `subfamilyName` 相同、`postscriptName` 不同 | `nameConflicts` (type `name`)       |
| 同字体多格式 | 内容指纹相同、`format` 不同                                    | `formatDuplicates` (type `format`)  |

> **建议**：以「PostScript 名 + 子家族名 + 版本 + 字形数 + 字符集摘要」的内容指纹作为主键，而非单纯的字符串比对。字符串比对会漏掉「PostScript 名相同但内容不同」这类情况。

### 4.8 `mfp-preview.js`

```js
// 按模板渲染预览图
export async function renderPreview(template, fontBuffer, metadata, options)
//   options: { environment: 'browser' | 'node', fontFaceAlias?: string }
//   → Buffer（PNG）

// 替换模板变量（规范 4.5.2）
export function interpolateTemplate(text, metadata)  // → string

// 按回退链决定预览来源（规范 4.4.6）
export async function resolvePreview(fontEntry, archiveHandle, options)
//   → { source: 'user' | 'generated' | 'embedded' | 'fallback', image?: Buffer, text?: string }

// 内置默认模板（规范 4.5.1 的结构）
export function defaultPreviewTemplate()  // → Template 对象

// 校验模板是否符合约束（规范 4.5.3：只支持单字体、画布尺寸上限）
export function validateTemplate(template)  // → { valid, errors }
```

**渲染实现的两条路径**：

| 环境                    | 方案                     | 说明                                                                                  |
| --------------------- | ---------------------- | ----------------------------------------------------------------------------------- |
| 浏览器 / 渲染进程            | 2D Canvas + `FontFace` | 直接喂字体二进制给 `FontFace`（**不是** base64），`ctx.fillText` 后 `canvas.toBlob`                |
| Node.js（无原生 Canvas 时） | 字体引擎取字形路径 + `Path2D`   | `font.layout(text)` → 逐字形取路径 → 用 `Path2D` 绘制；**兜底**：只渲染纯文本回退（规范 4.4.6 的 `fallback`） |

### 4.9 `mfp-install.js`

```js
// 从容器提取字体并安装（调用宿主安装接口）
export async function installFromArchive(session, fontIds, options)
//   options: { scope: 'user' | 'system', onProgress? }
//   → { installed: [], duplicates: [], failures: [] }

// 仅提取到指定目录，不安装
export async function extractFromArchive(session, fontIds, targetDirectory, options)
//   → { extracted: [{ fontId, filePath }] }

// 清理临时提取目录
export async function cleanupExtracted(extractedList)  // → void
```

**`installFromArchive`** **的步骤**：

```
1. 校验 fontIds 全部属于该 session 的 manifest
2. 创建临时目录：join(tmpdir(), `mfp-extract-${sessionId}-${Date.now()}`)
3. 逐个字体：
   a. 从容器流式读取条目 → 写入临时文件
      · 条目名与扩展名由 manifest.file 决定，保留原格式（ttc 仍是 ttc）
   b. 若 scope === 'system' → 走宿主系统级安装（需提权，见 9.7）
      否则 → 调用宿主用户级安装接口
   c. 收集返回值：
      · 成功                → installed
      · 已存在（duplicate） → duplicates（附匹配信息，与 manifest.conflicts 合并呈现）
      · 抛错                → failures
   d. 上报进度 onProgress({ phase: 'install', current, total })
4. 清理临时目录（finally 中执行，确保异常时也清理）
5. 返回汇总
```

**约束**：

- **必须**先提取到磁盘再调用宿主安装接口——绝大多数平台的字体安装 API 只接受文件路径
- **必须不**把 ZIP 流直接喂给安装接口
- 临时目录**必须**在 `finally` 中清理
- TTC/OTC **必须**整体安装（规范 4.4.4：`installMode: 'allFaces'`），**必须不**拆分 face

### 4.10 `mfp-session.js`

```js
// 会话级授权
export function getAuthorization(callerId)
//   → { opened: Map<sessionId, Session>, writeTargets: Set<path> }

export function assertSessionOwned(callerId, sessionId)  // → Session
export function assertWriteAuthorized(callerId, path)    // → normalizedPath
export function clearAuthorization(callerId)             // → void

// 打开容器并建立会话
export async function openSession(callerId, filePath, options)
//   → { sessionId, manifest, resolvedFonts, hasRecovery }

// 关闭会话，释放句柄
export async function closeSession(callerId, sessionId)  // → void
```

**授权模型**：

- 按 `callerId` 隔离，不同调用方（窗口 / 进程）互不可见
- 只有经宿主文件选择对话框授权过的路径才可读写
- 调用方销毁时清理全部会话
- 会话内缓存的 `ArchiveHandle` **必须**在关闭或调用方销毁时 `close()`

***

## 5. 宿主环境接口契约

本章给出核心库对宿主暴露的接口清单。**接口名与传输机制由实现者按平台选择**——桌面应用可用 IPC 频道，CLI 可直接调用函数，Web 端可用 Worker 消息。

### 5.1 接口清单

| 接口                 | 方向          | 参数                                     | 返回                                                                                  |
| ------------------ | ----------- | -------------------------------------- | ----------------------------------------------------------------------------------- |
| `open`             | UI → 核心     | —                                      | `{ sessionId, path, manifest, hasRecovery }`                                        |
| `authorizeOpen`    | UI → 核心     | `filePath`                             | 同上（拖拽/命令行传入时用）                                                                      |
| `close`            | UI → 核心     | `sessionId`                            | `{ ok }`                                                                            |
| `listFonts`        | UI → 核心     | `sessionId`                            | `fonts[]`（含 `installed` 标记与 `conflicts` 标记）                                         |
| `previewImage`     | UI → 核心     | `sessionId, fontId`                    | 位图数据（PNG）                                                                           |
| `previewData`      | UI → 核心     | `sessionId, fontId, faceIndex`         | 字体二进制（供 `FontFace` 加载）                                                              |
| `extract`          | UI → 核心     | `sessionId, fontIds, targetDirectory?` | `{ extracted: [{ fontId, filePath }] }`                                             |
| `install`          | UI → 核心     | `sessionId, fontIds, scope`            | `{ installed, duplicates, failures }`                                               |
| `verify`           | UI → 核心     | `sessionId, fontIds?`                  | `{ results: [{ entry, ok, expected, actual }] }`                                    |
| `selectPackTarget` | UI → 核心     | —                                      | `outputPath`                                                                        |
| `pack`             | UI → 核心     | `options`                              | `{ outputPath, revision, fontCount }`                                               |
| `appendFonts`      | UI → 核心     | `sessionId, filePaths[], options`      | `{ revision, added, duplicates }`                                                   |
| `removeFonts`      | UI → 核心     | `sessionId, fontIds[]`                 | `{ revision, removed }`                                                             |
| `compact`          | UI → 核心     | `sessionId`                            | `{ beforeBytes, afterBytes, revision }`                                             |
| `storageState`     | UI → 核心     | `sessionId`                            | `{ totalBytes, garbageBytes, garbageRatio, threshold, entryCount, liveEntryCount }` |
| `reveal`           | UI → 核心     | `filePath`                             | `{ ok }`                                                                            |
| `progress`         | 核心 → UI（推送） | —                                      | `{ sessionId, phase, current, total }`                                              |

**可能返回的错误码**（规范 10.3）：

| 接口                                                                             | 可能错误码                                                                                                   |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `open` / `authorizeOpen`                                                       | `E_MAGIC` `E_FORMAT_VERSION` `E_HEADER_CRC` `E_ZIP_STRUCTURE` `E_MANIFEST_MISSING` `E_MANIFEST_INVALID` |
| `close` / `listFonts` / `previewImage` / `previewData` / `extract` / `install` | `E_ENTRY_MISSING`                                                                                       |
| `verify`                                                                       | `E_CHECKSUM_MISMATCH`                                                                                   |
| `pack`                                                                         | `E_THRESHOLD_RANGE`                                                                                     |
| `appendFonts`                                                                  | `E_LOCKED` `E_COMPACTION_NEEDED`                                                                        |
| `removeFonts` / `compact`                                                      | `E_LOCKED`                                                                                              |

### 5.2 事件与进度推送

`phase` 取值：`open` / `parse` / `preview` / `pack` / `extract` / `install` / `compact` / `verify`。

推送载荷为 `{ sessionId, phase, current, total }`。订阅接口**应当**返回取消订阅的函数，避免监听器泄漏。

**错误约定**：核心库抛出的错误**必须**带 `errorCode` 属性（规范 10.3 的错误码），UI 层据此区分处理与提示。

### 5.3 会话授权模型

核心库对文件系统的访问**必须**受会话授权约束：

```
getAuthorization(callerId) → { opened: Map<sessionId, Session>, writeTargets: Set<path> }
```

- 按 `callerId` 隔离，不同调用方互不可见
- 只有经文件选择对话框授权过的路径才可读写
- `assertWriteAuthorized(callerId, path)` 在每次写入前校验并返回规范化后的路径
- 调用方销毁时 `clearAuthorization(callerId)`，并 `close()` 其中缓存的全部 `ArchiveHandle`

**理由**：拖拽或命令行传入的路径未经对话框授权。若不设白名单，UI 层一旦被注入即可让核心库读写任意文件。

### 5.4 持久化键

| 键名                     | 用途                        |
| ---------------------- | ------------------------- |
| `mfp-last-pack`        | 最近打开的容器路径                 |
| `mfp-install-scope`    | 默认安装范围（`user` / `system`） |
| `mfp-preview-template` | 当前选用的预览模板 id              |
| `mfp-list-mode`        | 包内字体列表视图模式                |
| `mfp-compaction-mode`  | 紧凑化策略（`auto` / `manual`）  |

主进程侧偏好落盘到 `app-preferences/mfp.json`，**建议**采用原子写（写临时文件 + rename）范式，避免写入中断留下半截文件。

***

## 6. 核心流程与伪代码

### 6.1 打开与识别

> 对应 [`SPEC.md`](./SPEC.md) 第 8 节与 8.1 校验顺序第 1–5 步

```js
async function openSession(callerId, filePath, options) {
  // 1. 校验路径已授权
  const path = assertWriteAuthorized(callerId, filePath)

  // 2. 读前 32 字节
  const headerBuffer = await readRange(path, 0, MFP_HEADER_BYTES)

  // 3. 解析并校验 Header（magic / formatVersion / headerCRC32）
  //    规范 8.1 校验顺序第 1–3 步
  const header = decodeHeader(headerBuffer)   // 失败抛 E_MAGIC / E_FORMAT_VERSION / E_HEADER_CRC
  const flags = readFlags(header.flags)

  // 4. 打开 ZIP（只读中央目录）
  const handle = await openArchive(path)

  // 5. 校验 manifestOffset 处签名为 0x04034B50（规范 8.1 第 4 步）
  await assertLocalHeaderSignature(path, header.manifestOffset)

  // 6. 读并解析 manifest（内部做 Schema 校验）
  const { manifest } = await readManifest(handle)

  // 7. revision 链重建（历史缺失视为链终点，规范 4.2.3）
  const resolved = await resolveRevisionChain(handle, manifest)

  // 8. 检查 Footer（规范 8.1 第 8 步，可降级）
  const footer = flags.hasFooter
    ? await decodeFooter(handle, handle.eocdOffset)
    : null
  if (flags.hasFooter && !footer) {
    reportDiagnostic('E_ZIP_STRUCTURE', 'HAS_FOOTER 置位但未找到 Footer Descriptor')
  }

  // 9. 建立会话
  const sessionId = createSessionId()
  getAuthorization(callerId).opened.set(sessionId, {
    handle,
    path,
    manifest,
    fonts: resolved.fonts,
    templates: resolved.templates
  })

  return {
    sessionId,
    path,
    manifest,
    hasRecovery: flags.hasRecovery
  }
}
```

### 6.2 打包

> 对应 [`SPEC.md`](./SPEC.md) 3.3、3.3.4、4.5、4.8、10.2

```js
async function pack(callerId, options, progress) {
  // options: {
  //   fontPaths: string[], outputPath: string,
  //   package: { id, name, version, description?, tags? },
  //   template?: Template, previewText?: string,
  //   userPreviews?: Map<fontPath, imagePath>,
  //   allowRedistribution?: boolean, keepHistory?: boolean
  // }

  // 1. 解析字体元数据
  const fontEntries = []
  for (const [index, fontPath] of options.fontPaths.entries()) {
    progress({ phase: 'parse', current: index + 1, total: options.fontPaths.length })

    const buffer = await fs.readFile(fontPath)
    const container = fontEngine.create(buffer)
    const faces = buildFontFaces(container, fontPath)

    const sha256 = sha256Hex(buffer)
    const format = extensionOf(fontPath)
    const isCollection = faces.length > 1

    fontEntries.push({
      id: `font-${String(index + 1).padStart(3, '0')}`,
      hash: `sha256:${sha256}`,
      file: fontEntryName(sha256, format),
      originalName: basename(fontPath),
      format,
      isCollection,
      faceCount: faces.length,
      // 规范 4.4.4：集合字体必须 allFaces
      installMode: isCollection ? 'allFaces' : 'singleFace',
      faces: isCollection ? mapFaces(faces) : undefined,
      metadata: mapMetadata(faces[0], container),
      _buffer: buffer,   // 仅内存中使用，不写入 manifest
      _faces: faces
    })
  }

  // 2. 去重与冲突检测（规范 4.8）
  const { kept, skipped } = deduplicateFonts(fontEntries)
  const conflicts = detectConflicts(kept, [])

  // 3. 渲染预览图（规范 4.5）
  for (const entry of kept) {
    progress({ phase: 'preview', current: ..., total: kept.length })
    const template = options.template || defaultPreviewTemplate()
    const preview = options.userPreviews?.has(entry.originalName)
      ? { source: 'user', image: await fs.readFile(options.userPreviews.get(entry.originalName)) }
      : { source: 'generated', image: await renderPreview(template, entry._buffer, entry.metadata, { environment: 'browser' }) }
    entry.preview = {
      image: previewEntryName(entry.hash.slice(7), 0),
      templateId: template.templateId,
      generatedText: extractSampleText(template),
      source: preview.source
    }
    entry._previewImage = preview.image
  }

  // 4. 组装 manifest（revision = 1）
  const manifest = buildManifest({
    package: options.package,
    fonts: kept.map(stripInternalFields),
    templates: [{ id: 'default-v1', name: '默认预览模板', isDefault: true, file: templateEntryName('default-v1') }],
    conflicts,
    storage: { compactionThreshold: 0.2, lastCompactionRevision: 1, entryCount: ..., liveEntryCount: ... },
    revision: 1,
    parentRevision: null,
    operation: 'create',
    changes: { added: kept.map((f) => f.id), removed: [], modified: [], compacted: false }
  })

  // 5. 创建 ZIP64 归档（规范 3.3）
  const writer = createArchiveWriter(options.outputPath)

  // 5.1 manifest 必须是第一个条目（规范第 7 节约束 3）
  //     manifestHash 先占位，最后回填
  writer.appendEntry('manifest.json', Buffer.from(JSON.stringify(manifest)), { method: 'deflate' })

  // 5.2 字体条目（规范 10.2：字体用 store）
  for (const entry of kept) {
    writer.appendEntry(entry.file, entry._buffer, { method: 'store' })
  }

  // 5.3 预览图、模板、许可证
  for (const entry of kept) {
    if (entry._previewImage) writer.appendEntry(entry.preview.image, entry._previewImage, { method: 'store' })
  }
  writer.appendEntry(templateEntryName('default-v1'), Buffer.from(JSON.stringify(options.template || defaultPreviewTemplate())), { method: 'deflate' })

  await writer.finalize()

  // 6. 前置 MFP Header 并修正 ZIP 偏移（规范 3.3.4，关键步骤）
  await prependHeaderAndPatchOffsets(options.outputPath, {
    formatVersion: 1,
    flags: writeFlags({ hasRecovery: false, hasFooter: false, hasHistory: false }),
    createdAt: Date.now(),
    // manifest 是第一个条目，紧邻 Header 之后
    manifestOffset: MFP_HEADER_BYTES
  })

  // 7. 回填 checksum 与 manifestHash，重写 manifest 条目
  //    注意：重写 manifest 会改变其大小 → 必须走「替换 manifest」路径而非原地改
  await rewriteManifestWithChecksum(options.outputPath, manifest)

  return { outputPath: options.outputPath, revision: 1, fontCount: kept.length, skipped: skipped.length }
}
```

**顺序约束**：Header **必须**在 ZIP 结构完成之后再前置并修正偏移（第 6 步），因为修正依赖完整可解析的中央目录。`checksum` 的回填（第 7 步）会让 manifest 尺寸变化，因此**必须**通过 `replaceManifest` 重新走一次追加流程，而**必须不**在原地覆盖字节。

### 6.3 增量追加

> 对应 [`SPEC.md`](./SPEC.md) 4.2、4.6

```js
async function appendFonts(callerId, sessionId, filePaths, options) {
  const session = assertSessionOwned(callerId, sessionId)

  // 1. 先判断是否需要紧凑化（规范 4.6）
  const context = await buildCompactionContext(session.path)
  const { needed, ratio, threshold } = await shouldCompact(session.path, context)
  if (needed && options.autoCompact !== false) {
    await compact(session.path)
  } else if (needed) {
    throw new MfpError('E_COMPACTION_NEEDED', { ratio, threshold })
  }

  // 2. 解析新字体（同 6.2 第 1 步）
  const newEntries = await parseFontsForAppend(filePaths)

  // 3. 冲突检测（与容器内已有字体比对）
  const conflicts = detectConflicts(newEntries, session.fonts)
  const { kept } = deduplicateFonts(newEntries)

  // 4. 渲染预览图
  for (const entry of kept) { /* 同 6.2 第 3 步 */ }

  // 5. 构造新 manifest（revision + 1）
  const newManifest = {
    ...session.manifest,
    revision: session.manifest.revision + 1,
    parentRevision: session.manifest.revision,
    updatedAt: new Date().toISOString(),
    operation: 'addFont',
    changes: { added: kept.map((f) => f.id), removed: [], modified: [], compacted: false },
    fonts: [...session.manifest.fonts, ...kept.map(stripInternalFields)],
    conflicts: mergeConflicts(session.manifest.conflicts, conflicts)
  }

  // 6. 估算写入耗时（用于动态 stale）
  const estimatedMs = estimateAppendMs(kept)

  // 7. 执行追加（内部加锁、截断、重写 CD、重写 Header）
  const result = await appendEntries(session.path, [
    ...kept.map((f) => ({ name: f.file, source: f._buffer, method: 'store' })),
    ...kept.filter((f) => f._previewImage).map((f) => ({ name: f.preview.image, source: f._previewImage, method: 'store' })),
    { name: 'manifest.json', source: Buffer.from(JSON.stringify(newManifest)), method: 'deflate' }
  ], {
    newManifest,
    estimatedDurationMs: estimatedMs
  })

  // 8. 刷新会话内的 manifest 缓存
  session.manifest = newManifest
  session.fonts = [...session.fonts, ...kept]

  return { revision: result.revision, added: kept.length, duplicates: conflicts.duplicates }
}
```

**为什么 manifest 条目在追加时会「多出一份」**：ZIP 允许同名条目存在多份，读取时以**中央目录中最后出现的那份**为准。`appendEntries` 在重建中央目录时会保留旧条目记录（其数据成为垃圾数据），因此读取侧自然拿到新的 manifest。旧 manifest 数据在紧凑化时被回收。

### 6.4 紧凑化

> 对应 [`SPEC.md`](./SPEC.md) 4.6

```js
async function compact(filePath, options) {
  const temporaryPath = `${filePath}.compact-${Date.now()}.tmp`

  // 1. 加锁（紧凑化耗时可能很长，stale 需按文件大小动态放大）
  const stat = await fs.stat(filePath)
  const estimatedMs = estimateCompactMs(stat.size)
  const release = await acquireLock(filePath, { estimatedDurationMs: estimatedMs })

  try {
    // 2. 打开旧容器
    const source = await openArchive(filePath)

    // 3. 读 manifest，确定全部有效条目
    const { manifest } = await readManifest(source)
    const liveEntries = collectLiveEntryNames(manifest)   // manifest.json + fonts/* + previews/* + templates/* + ...

    // 4. 写临时文件：只写有效条目
    const writer = createArchiveWriter(temporaryPath)
    for (const name of liveEntries) {
      progress({ phase: 'compact', current: ..., total: liveEntries.length })
      await refreshLock(release, filePath)   // 长耗时操作必须续期
      const buffer = await source.readEntryBuffer(name)
      writer.appendEntry(name, buffer, { method: compressionFor(name) })
    }
    await writer.finalize()

    // 5. 新 manifest：revision + 1，operation = 'compact'，storage 重置
    const newManifest = {
      ...manifest,
      revision: manifest.revision + 1,
      parentRevision: manifest.revision,
      updatedAt: new Date().toISOString(),
      operation: 'compact',
      changes: { added: [], removed: [], modified: [], compacted: true },
      storage: { ...manifest.storage, garbageBytes: 0, garbageRatio: 0, lastCompactionRevision: manifest.revision + 1, entryCount: liveEntries.length, liveEntryCount: liveEntries.length }
    }
    // 重写 manifest 条目（同上：替换而非原地改）
    await rewriteManifestInFile(temporaryPath, newManifest)

    // 6. 前置 Header 并修正偏移
    await prependHeaderAndPatchOffsets(temporaryPath, { /* 同 6.2 第 6 步 */ })

    // 7. fsync + 原子替换
    await fsyncFile(temporaryPath)
    await fs.rename(temporaryPath, filePath)   // 同分区 rename 是原子的

    await source.close()
    return { newRevision: newManifest.revision, beforeBytes: stat.size, afterBytes: (await fs.stat(filePath)).size }
  } finally {
    // 失败时清理临时文件
    await fs.rm(temporaryPath, { force: true }).catch(() => {})
    await release()
  }
}
```

### 6.5 安装

> 对应 [`SPEC.md`](./SPEC.md) 4.4.4

```js
async function installFromArchive(session, fontIds, options) {
  const installed = []
  const duplicates = []
  const failures = []
  const extracted = []
  const temporaryDirectory = join(tmpdir(), `mfp-extract-${session.sessionId}-${Date.now()}`)

  try {
    await fs.mkdir(temporaryDirectory, { recursive: true })

    for (const [index, fontId] of fontIds.entries()) {
      const font = session.fonts.find((f) => f.id === fontId)
      if (!font) { failures.push({ fontId, error: 'E_ENTRY_MISSING' }); continue }

      options.onProgress?.({ phase: 'extract', current: index + 1, total: fontIds.length })

      // 1. 流式提取到临时文件（保留原格式，TTC 不拆分）
      const temporaryFile = join(temporaryDirectory, font.originalName || basename(font.file))
      await extractEntryToFile(session.handle, font.file, temporaryFile)
      extracted.push(temporaryFile)

      // 2. 按范围安装
      try {
        if (options.scope === 'system') {
          // 需提权，见 9.7
          await installToSystem(temporaryFile, font)
        } else {
          // 宿主用户级安装：只接受磁盘路径
          const result = await hostInstallFont(temporaryFile)
          if (result.ok) {
            installed.push({ fontId, path: result.installedPath, provider: result.provider })
          } else if (result.duplicate) {
            duplicates.push({ fontId, matches: result.matches })
          } else {
            failures.push({ fontId, error: result.error || '安装失败' })
          }
        }
        options.onProgress?.({ phase: 'install', current: index + 1, total: fontIds.length })
      } catch (error) {
        failures.push({ fontId, error: error.message })
      }
    }

    return { installed, duplicates, failures }
  } finally {
    // 无论成功失败都清理临时文件
    await cleanupExtracted(extracted)
    await fs.rm(temporaryDirectory, { recursive: true, force: true }).catch(() => {})
  }
}
```

### 6.6 预览渲染

> 对应 [`SPEC.md`](./SPEC.md) 4.4.6、4.5

```js
// 包内字体预览的加载逻辑
async function loadPreviewFace(sessionId, fontId, faceIndex, alias) {
  // 1. 取字体二进制（走 previewData 接口，不落盘）
  const buffer = await host.previewData(sessionId, fontId, faceIndex)

  // 2. 注册 FontFace（直接喂 Uint8Array，不是 base64）
  const face = new FontFace(alias, new Uint8Array(buffer), { style: 'normal', weight: '400', stretch: 'normal' })
  await face.load()
  document.fonts.add(face)

  return alias
}

// 预览图：优先用包内图片，缺失时回退生成
async function resolvePreviewImage(sessionId, font) {
  if (font.preview?.image) {
    const image = await host.previewImage(sessionId, font.id)
    if (image?.byteLength) return { kind: 'image', data: image }
  }
  // 回退：用 FontFace 渲染纯文本（规范 4.4.6 的 fallback）
  const alias = await loadPreviewFace(sessionId, font.id, 0, `mfp-${font.id}`)
  return { kind: 'text', alias, text: font.metadata.fullName }
}
```

**约束**：包内字体预览**必须**走独立的 `previewData` 接口，**必须不**复用「只读已安装字体」的接口——后者的路径白名单不含包内未安装字体，会被拒绝。

**预览资源管理建议**：包内字体可能成百上千，**建议**对 `FontFace` 做懒加载与淘汰（例如数量上限 60、字节预算 256 MB），并在列表滚动时用 `IntersectionObserver` 或等价机制按需加载。

### 6.7 元数据字段映射

manifest 的 `metadata` 字段来源对照：

| manifest 字段                  | 来源                               | 说明                            |
| ---------------------------- | -------------------------------- | ----------------------------- |
| `familyName`                 | `name` 表 nameID 1                | 可做本地化（中文环境优先取本地化名称）           |
| `subfamilyName`              | `name` 表 nameID 2                | <br />                        |
| `fullName`                   | `name` 表 nameID 4                | <br />                        |
| `postscriptName`             | `name` 表 nameID 6                | <br />                        |
| `version`                    | `name` 表 nameID 5                | <br />                        |
| `copyright`                  | `name` 表 nameID 0                | <br />                        |
| `designer`                   | `name` 表 nameID 9                | <br />                        |
| `designerURL`                | `name` 表 nameID 12               | <br />                        |
| `license`                    | `name` 表 nameID 13               | 同时映射到 `license.type`          |
| `licenseURL`                 | `name` 表 nameID 14               | <br />                        |
| `characterSet.glyphCount`    | 字形总数                             | <br />                        |
| `characterSet.unicodeRanges` | 字符集压缩为区间                         | 输出 `U+XXXX` 或 `U+XXXX-U+YYYY` |
| `variationAxes`              | `fvar` 表                         | 无则填 `null`                    |
| `colorTables`                | `COLR` / `CPAL` / `sbix` 等表存在性检查 | 输出表名数组                        |
| `availableFormats`           | 打包时汇总                            | 同一字体的多种格式                     |
| `woff2File`                  | 打包时汇总                            | 同字体的 WOFF2 条目名                |

**不写入 manifest 的字段**：字重/字宽/倾斜等由平台在安装后自行判定的属性，以及仅用于打包期冲突检测的内容指纹。

**约束**：**必须**复用同一套字体解析逻辑产出 manifest 元数据与客户端字体列表元数据，否则两处会出现口径差异（例如家族名的本地化规则不一致，导致同一字体在两个界面显示不同名字）。

***

## 7. 依赖建议

以下为 JS / Node.js 生态的参考实现依赖。

| 包                 | License | 用途          | 必需性              | 替代方案                                     | 体积影响          |
| ----------------- | ------- | ----------- | ---------------- | ---------------------------------------- | ------------- |
| `archiver`        | MIT     | 流式 ZIP64 创建 | **必需**           | `zip-writer`（更小、Web Streams API）         | 约 1.5 MB      |
| `unzipper`        | MIT     | 随机访问 + 流式读取 | **必需**           | `@zokugun/yauzl-plus`（ZIP64 覆盖更严格）       | 约 500 KB      |
| `proper-lockfile` | MIT     | 跨进程文件锁      | **必需**           | 自研基于 `mkdir` 的锁                          | 约 50 KB       |
| `sudo-prompt`     | MIT     | Windows 提权  | **可选**（仅系统级安装需要） | 打包后 exe 自提权                              | 约 30 KB       |
| 字体引擎（如 `fontkit`） | MIT     | 字体元数据解析     | **必需**           | 各语言均有等价库                                 | 约 1 MB        |
| 原生 `canvas`       | MIT     | 打包侧 PNG 渲染  | **可选**           | 渲染进程用浏览器 Canvas；Node 侧用字体引擎路径 + `Path2D` | 原生模块，约 10 MB+ |

**关于原生** **`canvas`**：它是原生模块，会引入编译产物与平台相关二进制。若打包功能内置在桌面客户端的渲染进程中运行（而非独立 Node CLI），则**不需要**它——直接用浏览器 Canvas 即可。**建议**第一版打包功能内置在客户端内，避免引入原生依赖。

**关于传递依赖**：某些包可能作为其他工具的传递依赖出现在 `node_modules` 中，但**必须不**直接 `require`——传递依赖不保证版本与存在性。**必须**在依赖清单中显式声明。

***

## 8. 打包侧与客户端侧的能力划分

| 能力    | 打包侧（生成 .mfp）            | 客户端侧（读取 .mfp）                           |
| ----- | ----------------------- | --------------------------------------- |
| 字体解析  | 必须                      | 可选（仅在 manifest 缺 `faces` 时补解析）          |
| 冲突检测  | 必须（写入 `conflicts`）      | 可选（读取时二次校验）                             |
| 预览图渲染 | 优先（写入 `previews/*.png`） | 回退（`source` 为 `fallback` 或 `image` 缺失时） |
| 校验和   | 必须（写入 `checksum`）       | 可选（`verifyChecksum`）                    |
| 恢复记录  | 可选（P3 阶段）               | 可选（修复逻辑）                                |
| 增量更新  | 可以（同一套 `mfp-append.js`） | 可以                                      |
| 紧凑化   | 可以                      | 可以                                      |
| 安装    | 不涉及                     | 必须                                      |
| 字体注册  | 不涉及                     | 必须（系统级还需提权）                             |

**共享模块**：`mfp-format.js`、`mfp-manifest.js`、`mfp-archive.js`、`mfp-lock.js`、`mfp-append.js`、`mfp-compact.js`、`mfp-conflict.js` 在打包侧与客户端侧**完全共用**，无环境差异。仅 `mfp-preview.js` 有环境分支（`environment: 'browser' | 'node'`）。

***

## 9. 风险与边界

### 9.1 无法阻止解压

底层是标准 ZIP64，通用工具能读到明文中央目录与全部条目名（规范 3.3.3）。这是**刻意的设计选择**（规范 1.5），不是缺陷。**必须**在需求沟通阶段对齐：需要隐藏条目名的场景不适用本格式。

### 9.2 偏移修正的实现风险

规范 3.3.4 要求修正 5 处字段。漏改任何一处都会让通用工具解压失败，且**症状隐蔽**——MFP 客户端自己可能仍能正常读取（因为它走 `manifestOffset` 而非中央目录）。

**建议**在打包流程末尾加一步自检：用独立于写入路径的 ZIP 读取库打开生成的文件、列出全部条目并实际解压其中一个，验证偏移修正正确。

### 9.3 大文件的安全边界

- 所有偏移**必须**用 64 位整数
- 宿主 API 若只接受普通整数，转换前**必须**校验 `<= Number.MAX_SAFE_INTEGER`
- 单次内存分配有上限（JS 中约 2 GB）——**必须**用流式读写，**必须不**把整个字体条目一次性读入内存后再判断大小
- **建议**对单个条目设置上限（如 512 MB），超过则走分块流式路径

### 9.4 并发写入

**所有**读写容器的路径**必须**统一经 `mfp-lock.js`。任何绕过锁的访问（例如直接用 ZIP 读取库打开正在被追加的文件）都会破坏锁语义，可能读到中间状态。

### 9.5 崩溃一致性

增量追加的「截断」到「重写 Header」之间若进程崩溃，容器处于不一致状态。缓解措施：

1. Header 的 `headerCRC32` 作为「容器完整」的最后标记——Header 写完才代表操作完成
2. 读取侧发现 Header CRC 失败时，**应当**提示用户容器可能损坏，并建议用恢复记录修复
3. 恢复记录（P3 阶段）是唯一能真正修复的手段

### 9.6 预览渲染的降级路径

| 环境                  | 方案                                   | 限制                       |
| ------------------- | ------------------------------------ | ------------------------ |
| 浏览器 / 渲染进程          | Canvas + `FontFace` + `ctx.fillText` | 需在渲染进程中执行，核心库需通过宿主接口协调   |
| Node.js（无原生 Canvas） | 字体引擎 `layout()` + 字形路径 + `Path2D`    | 需自行处理字形定位与描边/填充；复杂排版支持有限 |
| 兜底                  | 纯文本回退（规范 4.4.6 的 `fallback`）         | 无图片，仅显示字体名               |

### 9.7 宿主安装接口的行为差异

| 议题               | 说明                                                                                                                                                                                                                                                                      |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **只接受磁盘路径**      | 绝大多数平台的字体安装 API 第一步就做 `exists` + `stat` 校验，因此**必须**先提取到临时文件                                                                                                                                                                                                             |
| **用户级 vs 系统级**   | 用户级安装通常无需提权（写用户字体目录 + 用户级注册表项）；系统级需要提权，且 `AddFontResourceEx` 之类的调用只对当前会话有效，**永久安装必须复制文件 + 写注册表**                                                                                                                                                                        |
| **集合字体的注册表写法**   | TTC/OTC 整体安装时，平台需要为集合内**每个 face 各注册一条**，格式为 `"<FamilyName> (TrueType)" = "<filename>.ttc"`。注意多个 face 可能有相同的 familyName（如 "My Font Regular" 与 "My Font Bold" 的 familyName 都是 "My Font"），此时注册表值名会冲突，只有后写入的生效。**建议**值名使用 `"<familyName> <subfamilyName> (TrueType)"` 以保证唯一 |
| **用户级字体的旧应用兼容性** | 用户字体目录是较新的系统机制，部分旧应用（尤其 Java 应用）只查系统字体目录。**建议**在安装范围选择处提示用户                                                                                                                                                                                                             |
| **查重的两个层次**      | `manifest.conflicts` 检查容器**内**字体之间；宿主安装接口的返回值检查容器内字体 vs **系统已安装**字体。**建议**在 UI 中分开展示「包内有重复」与「本机已安装同款」                                                                                                                                                                   |
| **批量安装的开销**      | 部分安装实现会在首次调用时触发一次全量字体枚举，之后走内存索引。批量安装时**建议**先预热一次，避免逐条触发                                                                                                                                                                                                                 |

### 9.8 协议版本升级

规范第 6 章规定：删除或重命名必需字段**必须**提升 `formatVersion` 主版本号。客户端**必须**在读取时校验版本（规范 8.1 第 2 步），**必须不**尝试读取高于自身支持版本的文件。

***

## 10. 分阶段落地建议

### P0：只读 + 安装（最小可用）

| 模块                | 内容                                             |
| ----------------- | ---------------------------------------------- |
| `mfp-format.js`   | Header/Footer 编解码、CRC32、偏移修正的**读取侧**（`-32` 换算） |
| `mfp-archive.js`  | 只读部分（ZIP 读取库封装）                                |
| `mfp-manifest.js` | manifest 读取 + Schema 校验                        |
| `mfp-session.js`  | 会话与授权                                          |
| `mfp-install.js`  | 提取 + 调用宿主安装接口（仅用户级）                            |
| UI                | 字体包页面、包内字体列表、页面状态                              |

**不含**：打包、增量更新、紧凑化、恢复记录。

**可验证**：打开一个 .mfp → 看到字体列表与预览 → 安装到用户级。

### P1：打包

| 模块                | 内容                                   |
| ----------------- | ------------------------------------ |
| `mfp-archive.js`  | 写入部分（ZIP 写入库封装）                      |
| `mfp-format.js`   | Header/Footer 编码、偏移修正的**写入侧**（`+32`） |
| `mfp-conflict.js` | 去重与冲突检测                              |
| `mfp-preview.js`  | 模板渲染（浏览器 Canvas）                     |
| UI                | 预览模板编辑器、打包入口                         |

**可验证**：选一批字体 → 生成 .mfp → 用 P0 的读取路径打开 → 用 7-Zip 解压验证条目可读且偏移正确。

### P2：增量更新 + 紧凑化

| 模块               | 内容           |
| ---------------- | ------------ |
| `mfp-lock.js`    | 并发锁与动态 stale |
| `mfp-append.js`  | 增量追加         |
| `mfp-compact.js` | 垃圾回收与动态阈值    |

**可验证**：追加字体后 `revision` 递增且旧字体仍可用 → 触发紧凑化后文件体积下降。

### P3：恢复记录 + 系统级安装

| 模块               | 内容             |
| ---------------- | -------------- |
| `mfp-format.js`  | Footer 的恢复记录读写 |
| `mfp-install.js` | 系统级安装（提权）      |

**可验证**：人为损坏中央目录后用恢复记录修复 → 系统级安装后字体出现在系统字体列表。

***

## 11. 验收清单

### 11.1 格式一致性

- [ ] 生成的容器前 4 字节为 `4D 46 50 01`，`headerCRC32` 校验通过
- [ ] `manifestOffset` 指向 `manifest.json` 的 Local File Header（签名 `0x04034B50`）
- [ ] **用 7-Zip / WinRAR 打开容器，能列出全部条目名且能成功解压**（验证规范 3.3.4 的 5 处偏移修正）
- [ ] 解出的字体文件能被系统正常安装
- [ ] `manifest.json` 是第一个条目
- [ ] 中央目录与 EOCD 结构完整，ZIP64 EOCD Record 与 Locator 均存在
- [ ] 条目名以 UTF-8 编码且置位 EFS 标志（bit 11）

### 11.2 读取

- [ ] 非 MFP 文件返回 `E_MAGIC`
- [ ] `formatVersion` 过高返回 `E_FORMAT_VERSION`
- [ ] 篡改 Header 后返回 `E_HEADER_CRC`
- [ ] manifest 结构非法返回 `E_MANIFEST_INVALID`
- [ ] 校验和不匹配返回 `E_CHECKSUM_MISMATCH` 且用户可选择继续
- [ ] 缺失 `templates` / `storage` / `conflicts` / `recovery` 时按规范 6.3 正常降级
- [ ] `parentRevision` 指向的历史条目缺失时不报错（规范 4.2.3）
- [ ] 校验顺序与规范 8.1 一致（同一损坏文件每次报同一个错误码）

### 11.3 打包

- [ ] manifest 元数据与客户端字体列表的元数据一致（同一字体在两处显示的 familyName / subfamilyName / postscriptName 相同）
- [ ] 集合字体（TTC/OTC）`isCollection: true`、`installMode: 'allFaces'`、`faceCount` 正确
- [ ] 完全重复的字体被跳过并记入 `conflicts.duplicates`
- [ ] 预览图按模板生成，`preview.source` 正确标注
- [ ] 字体条目使用 `store`、文本条目使用 `deflate`（规范 10.2）
- [ ] 超过 4 GB 的容器能正常生成与读取

### 11.4 安装

- [ ] 单字体（TTF/OTF）用户级安装成功
- [ ] 集合字体整体安装后，全部 face 都出现在字体列表
- [ ] 安装已存在的字体时返回 `duplicates`，UI 分开展示「包内重复」与「本机已安装」
- [ ] 安装完成后广播系统字体变更消息，其他应用能立即看到新字体
- [ ] 临时提取目录在成功与失败两种情况下都被清理
- [ ] 系统级安装触发提权，安装后字体在系统字体目录

### 11.5 增量更新与紧凑化

- [ ] 追加字体后 `revision` 递增、`parentRevision` 指向旧值、`operation` 为 `addFont`
- [ ] 追加后原有字体仍可正常读取与安装
- [ ] 追加后 Header 的 `manifestOffset` 与 `headerCRC32` 已重写
- [ ] 并发写入时第二个进程得到 `E_LOCKED` 或按重试策略等待
- [ ] 长耗时操作期间锁被续期，不会被其他进程误判过期
- [ ] 紧凑化后 `garbageBytes` 归零、`lastCompactionRevision` 更新
- [ ] 紧凑化通过临时文件 + 原子 rename 完成，中途崩溃不损坏原文件
- [ ] `compactionThreshold` 始终落在 `[0.05, 0.50]`

### 11.6 安全边界

- [ ] UI 层无法直接拼接任意磁盘路径——所有文件操作经宿主接口
- [ ] 会话按调用方隔离，调用方销毁后会话与文件句柄被释放
- [ ] 只有经文件选择对话框授权过的路径可被写入
- [ ] 模板变量替换不执行任意代码（规范 4.5.3）
- [ ] 解压出的字体在安装前不被自动执行或被当作可执行内容处理

