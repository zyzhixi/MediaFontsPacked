# 最小文件结构（字节级示意）

> 对应 [`../SPEC.md`](../SPEC.md) 3.1、3.2、3.3.4 与 10.6.1
> 本文件描述一个最小 `.mfp` 容器在磁盘上的字节排布，供实现者对照调试。

## 1. 总体排布

一个不含恢复记录、不含历史修订的最小容器：

```
偏移 0
┌────────────────────────────────────────────────┐
│  MFP Header（固定 32 字节）                     │
├────────────────────────────────────────────────┤
│  Local File Header + manifest.json（deflate）   │  ← manifestOffset 指向此处
├────────────────────────────────────────────────┤
│  Local File Header + fonts/<sha256>.ttf（store）│
├────────────────────────────────────────────────┤
│  Central Directory                              │
├────────────────────────────────────────────────┤
│  ZIP64 End of Central Directory Record          │
├────────────────────────────────────────────────┤
│  ZIP64 End of Central Directory Locator         │
├────────────────────────────────────────────────┤
│  End of Central Directory Record                │
└────────────────────────────────────────────────┘ 文件末尾
```

因为 `manifest.json` 是第一个条目且紧跟 Header 之后，Header 的 `manifestOffset` 恒为 `32`。

## 2. MFP Header 逐字节

| 偏移 | 长度 | 字段               | 类型     | 最小容器取值                    |
| -- | -- | ---------------- | ------ | ------------------------- |
| 0  | 4  | `magic`          | bytes  | `4D 46 50 01`（`MFP\x01`）  |
| 4  | 2  | `formatVersion`  | uint16 | `01 00`                   |
| 6  | 2  | `flags`          | uint16 | `00 00`（三个标志位均置 0）        |
| 8  | 8  | `createdAt`      | uint64 | Unix 毫秒时间戳（小端）            |
| 16 | 4  | `headerCRC32`    | uint32 | 覆盖偏移 0–15 与 20–31，共 28 字节 |
| 20 | 4  | `reserved`       | uint32 | `00 00 00 00`             |
| 24 | 8  | `manifestOffset` | uint64 | `20 00 00 00 00 00 00 00`（= 32） |

`flags` 位定义（SPEC.md 3.2.1）：

| 位    | 名称             | 最小容器取值      |
| ---- | -------------- | ----------- |
| 0    | 保留             | `0`         |
| 1    | `HAS_RECOVERY` | `0`（无恢复记录）  |
| 2    | `HAS_FOOTER`   | `0`（无 Footer） |
| 3    | `HAS_HISTORY`  | `0`（不保留历史修订） |
| 4–15 | 保留             | `0`         |

若 `HAS_RECOVERY=1`、`HAS_FOOTER=1`、`HAS_HISTORY=1`，则 `flags` = `0b1110` = `0x000E`（见 SPEC.md 10.6.2）。

## 3. headerCRC32 的计算范围

```
参与计算的字节 = Header[0..15] + Header[20..31]      // 共 28 字节
                                                   // Header[16..19] 按 4 字节 0x00 参与
CRC32 参数    = 多项式 0xEDB88320（反射）、初值 0xFFFFFFFF、输出取反
```

参考计算（JS 风格伪代码）：

```js
const scratch = Buffer.alloc(28)
headerBuffer.copy(scratch, 0, 0, 16)    // 偏移 0–15
headerBuffer.copy(scratch, 16, 20, 32)  // 偏移 20–31
const crc = crc32(scratch)              // 偏移 16–19 保持为 0
headerBuffer.writeUInt32LE(crc, 16)
```

## 4. 偏移修正（关键步骤）

先把 ZIP 正常写出，再把 32 字节 Header 前置。前置后，ZIP 内部所有**相对文件起始**的偏移都错位了 32 字节，必须修补以下 5 处（SPEC.md 3.3.4）：

| 序号 | 字段                                                              | 修正    |
| -- | --------------------------------------------------------------- | ----- |
| 1  | 每个中央目录条目的 *relative offset of local header*                    | `+= 32` |
| 2  | 中央目录条目 ZIP64 扩展字段（header ID `0x0001`）中的 *local header offset* | `+= 32` |
| 3  | EOCD 的 *offset of start of central directory*                   | `+= 32` |
| 4  | ZIP64 EOCD 的 *offset of start of central directory*             | `+= 32` |
| 5  | ZIP64 EOCD Locator 的 *relative offset of ZIP64 EOCD*            | `+= 32` |

**不修正**的字段：条目数、中央目录大小、各条目的压缩/未压缩大小、CRC32。

修补时从文件尾部向前搜索签名：

```
0x06054B50  EOCD             → 修 offset of start of central directory
0x07064B50  ZIP64 EOCD Locator → 修 relative offset of ZIP64 EOCD
0x06064B50  ZIP64 EOCD       → 修 offset of start of central directory
```

> **为什么这一步不能省**：漏改会让 7-Zip / WinRAR 按中央目录寻址时落到错误位置、解压失败；而 MFP 客户端自己可能仍"看起来正常"（因为它走 `manifestOffset` 而非中央目录）。症状极其隐蔽，因此 SPEC.md 把它列为第 1 号关键约束。
>
> **自检建议**：打包流程末尾用**独立于写入路径**的 ZIP 读取库打开产物，列出全部条目并实际解压其中一个。

## 5. 完整容器的尾部

带恢复记录时，文件末尾追加 Footer（SPEC.md 3.4）：

```
┌────────────────────────────────────────────────┐
│  Recovery Record（长度 = recoveryLength）       │
├────────────────────────────────────────────────┤
│  Footer Descriptor（固定 24 字节）              │
│  ├─ footerMagic    4B   4D 46 50 FF（MFP\xFF）  │
│  ├─ recoveryLength 8B   uint64 小端             │
│  ├─ footerCRC32    4B   覆盖 magic+length+reserved │
│  └─ reserved       8B   全 0                    │
└────────────────────────────────────────────────┘ 文件末尾
```

`footerCRC32` 覆盖 `footerMagic`（4B）+ `recoveryLength`（8B）+ `reserved`（8B）共 20 字节，自身 4 字节按 `0x00` 参与。

定位方式：先找到 ZIP EOCD，检查其**紧接**位置是否为 `4D 46 50 FF`。

## 6. 一致性约束（本示例相关）

- `recovery.length` 必须等于 Footer 的 `recoveryLength`
- `recovery.present` 为 `true` 时 Header 的 `HAS_RECOVERY` 必须置 1
- `manifestOffset` 必须指向 `manifest.json` 的 Local File Header（签名 `0x04034B50`），不是数据区
- 每次修改容器（增量更新、紧凑化）后必须重写 `manifestOffset` 并重算 `headerCRC32`
