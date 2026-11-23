# 优化方案 v2｜USN Journal 变更追踪 + 内存压缩（实测校准版）

> 配套：`docs/PRD.md`（需求）、`docs/技术方案.md`（v1 实现）、`docs/骨架图.md`（架构）。
> 本文所有"实测"结论均来自当前机器（Windows 10.0.26300 x64，C/D/E 均为 NTFS，当前用户**非管理员**）。

---

## 0. 结论（先说能做什么、不能做什么）

| 问题 | 结论 |
|---|---|
| 能否用 USN Journal 替代 FileSystemWatcher？ | **本地 NTFS 卷可以（且更好）；NAS 映射盘不行。** USN 是 NTFS 驱动在**本机卷**上维护的日志，`Z:` 的变更发生在 NAS 自己的文件系统上，Windows 只能通过 SMB2 `CHANGE_NOTIFY`（也就是 `FileSystemWatcher` 的底层）拿到"通知"，拿不到"日志"，更无法补读历史。 |
| USN 最大价值是什么？ | 把本地侧的扫描成本从 **O(树大小)** 降到 **O(变化数)**，且**不丢事件**（有 USN 断点续读），并天然提供 **FileReferenceNumber**，使"改名/移动"判定变成 O(1) 而不用哈希比对。 |
| 代价是什么？ | 读取 USN **必须提权**（实测：非管理员 `GENERIC_READ` 打开卷 → `ERROR_ACCESS_DENIED(5)`），且 USN 是有限环形缓冲（本机 `MaximumSize=32MB`，写满会覆盖），必须持久化检查点并在失效时降级全量扫描。 |
| 内存能压到多少？ | 从 v1 的 **~70MB 稳态** 降到 **~25–35MB 稳态 / ~45MB 峰值**，并且**与文件数量无关（O(1)）**。关键不是 USN，而是"取消内存清单，扫描直接落 SQLite staging 表，用 SQL 做集合运算"。 |
| 两者关系 | 相互独立、可分别上线：**内存压缩（§5）收益更大、风险更低，应先行**；USN（§4）是可选加速器，用 feature flag 挂在监控层。 |

---

## 1. 实测证据（本机，可复现）

### 1.1 USN 期刊存在且参数已知

```console
$ fsutil usn queryjournal C:
Usn 日志 ID   : 0x01d8ec1ae6620ed2
第一个 Usn    : 0x0000000d5b800000      # FirstUsn：仍可读的最早记录
下一个 Usn    : 0x0000000d5d9df868      # NextUsn：检查点要存的就是它
最低有效 Usn  : 0x0000000000000000
最大 Usn      : 0x00000fffffff0000
最大大小      : 0x0000000002000000 (32.0 MB)   # 环形缓冲上限 → 会覆盖
分配增量      : 0x0000000000800000 (8.0 MB)
支持的记录版本: 2 - 4
写入范围异常  : 已禁用
```

**解读（直接决定设计）**：
- `NextUsn` 是我们每轮读取的起点，必须落库；
- `32MB` 上限意味着"两次轮询之间产生超过 32MB 的 USN 记录"就会丢历史 → 轮询间隔必须短（**1–2 秒**），并对 `FirstUsn > checkpoint` 做失效检测；
- 记录版本 2–4 都支持，用 **V2** 解析即可（V3/V4 仅多了 128 位 FRN，可选支持）。

### 1.2 权限门槛（决定部署形态）

```console
$ fsutil usn readjournal C:
错误 5: 拒绝访问。                          # ← 当前用户非管理员，读取被拒

# P/Invoke 实测（CreateFile("\\\\.\\C:") + DeviceIoControl）
open acc=0x0          -> 句柄有效，但 FSCTL_QUERY_USN_JOURNAL 返回 err=1 (ERROR_INVALID_FUNCTION)
open acc=0x80 (READ_ATTRIBUTES) -> 句柄有效，FSCTL 仍 err=1
open acc=0x81 / 0x80000000 (GENERIC_READ) -> 打开失败 err=5 (ERROR_ACCESS_DENIED)
```

**结论**：`FSCTL_QUERY_USN_JOURNAL` / `FSCTL_READ_USN_JOURNAL` / `FSCTL_ENUM_USN_DATA` 需要以**管理员或 LocalSystem** 身份打开卷句柄。这直接产生一个部署前提（见 §4.6 与 PRD Q6）。

### 1.3 环境本身的结论
- C/D/E 全为 NTFS → 本地侧**具备** USN 条件；
- 当前会话 `net use` 为空、无 `Z:` → 远端侧必须按 UNC 设计（USN 与 SMB 无关，此结论不变）。

---

## 2. 变更监控通道全对比（这是"能否参考 USN"的正面回答）

| 机制 | 能覆盖 | 丢事件风险 | 权限 | 单轮成本 | 本方案用途 |
|---|---|---|---|---|---|
| `FileSystemWatcher`（= `ReadDirectoryChangesW`） | 本地任意 FS **+ SMB 远端** | **中高**：服务端缓冲（最大 64KB）溢出、断线、休眠唤醒后丢；不补历史 | 普通用户 | 事件驱动，≈0 | 本地降级模式 / **远端唯一实时手段** |
| **USN Journal**（`FSCTL_READ_USN_JOURNAL`） | **仅本机 NTFS 卷** | **无**（有 usn 断点，可补读，只要未超 `FirstUsn`） | **管理员** | 每轮读增量，通常 < 几十 KB | **本地首选** |
| `FSCTL_ENUM_USN_DATA`（MFT 顺序枚举） | 仅本机 NTFS 卷 | — | 管理员 | 顺序读 MFT，比递归 `FindFirstFile` 快一个量级 | 本地**首轮建索引**（可选，v2.1） |
| 目录 mtime 剪枝 | 本地/远端 | 只对**结构变化**（增删改名）敏感，**内容原地修改不改父目录 mtime** | 普通用户 | 减少下钻 | 远端增量枚举的剪枝加速（须配合全量校验） |
| 周期全量元数据枚举 | 本地/远端 | 无（天然兜底） | 普通用户 | O(N) 网络往返（SMB 用 `LARGE_FETCH` 批量） | **兜底**，永不可省 |

> 一句话：**USN 能"补历史"，`ReadDirectoryChangesW` 只能"报当下"。** 所以本地用 USN 消除周期全量扫描；远端没有 USN，只能"通知 + 周期枚举"双保险。

---

## 3. 目标架构：两端不同策略（不对称设计）

```
                         ┌──────────────── 本地 NTFS ────────────────┐
                         │ monitor = USN?（自动探测：提权 + NTFS + 期刊有效）
   每秒/每2秒轮询 ──────▶ │  FSCTL_READ_USN_JOURNAL(startUsn=checkpoint)
                         │      └─ 得到「变化路径集合」（含 RENAME 成对记录）
                         │           └─ 仅对这些路径 stat（不做全树扫描！）
                         │ monitor = FSW?（无权限/非 NTFS/期刊失效）
                         │  ReadDirectoryChangesW + 每 5min 增量枚举兜底
                         └───────────────────┬───────────────────────┘
                                             ▼
                                  统一的 Job 队列（定长 ring 8k）
                                             │
                         ┌───────────────────┴───────────────────────┐
                         │                 远端 SMB                  │
                         │  ReadDirectoryChangesW（唯一实时手段）      │
                         │  + 每 5min 增量枚举（目录 mtime 剪枝）      │
                         │  + 每 10–60min 全量元数据校验（可配）       │
                         └───────────────────────────────────────────┘
                                             ▼
                        Scan(仅变更项) → staging 表 → SQL 归并 → Decide() → Execute
```

与 v1 的差别只有三点：**本地监听换成 USN（可选）、扫描范围从"全树"变成"变更集"、清单从内存搬到 SQLite**。

---

## 4. USN 实现细节（照着写就行）

### 4.1 常量与结构

```csharp
// FileSync.Host/Monitoring/Usn/NativeUsn.cs
internal static class NativeUsn
{
    public const uint FSCTL_QUERY_USN_JOURNAL = 0x000900f4;
    public const uint FSCTL_READ_USN_JOURNAL  = 0x000900bb;
    public const uint FSCTL_ENUM_USN_DATA     = 0x000900b3;

    [StructLayout(LayoutKind.Sequential)]
    public struct USN_JOURNAL_DATA_V2
    {
        public ulong UsnJournalID, FirstUsn, NextUsn, LowestValidUsn, MaxUsn,
                     MaximumSize, AllocationDelta, MinSupportedMajorVersion;
        public ushort MaxSupportedMajorVersion;
        public uint   Flags, RangeTrackChunkSize;
        public long   RangeTrackFileSizeThreshold;
    }

    [StructLayout(LayoutKind.Sequential)]
    public struct READ_USN_JOURNAL_DATA_V0
    {
        public long StartUsn; public uint ReasonMask, ReturnOnlyOnClose;
        public ulong Timeout, BytesToWaitFor;
    }

    // USN_RECORD_V2 是变长记录：前 60 字节固定头，FileOffset 处是 UTF-16 文件名
    [StructLayout(LayoutKind.Sequential, Pack = 1)]
    public struct USN_RECORD_V2_HEADER
    {
        public uint   RecordLength;        // 0
        public ushort MajorVersion;        // 4
        public ushort MinorVersion;        // 6
        public ulong  FileReferenceNumber;       // 8  ← 本机唯一文件 ID（改名不变）
        public ulong  ParentFileReferenceNumber; // 16
        public long   Usn;                 // 24
        public long   TimeStamp;           // 32
        public uint   Reason;              // 40 ← 变化原因位掩码
        public uint   SourceInfo;          // 44
        public uint   SecurityId;          // 48
        public uint   FileAttributes;      // 52
        public ushort FileNameLength;      // 56
        public ushort FileNameOffset;      // 58
    }
}
```

`Reason` 位（只用这几个）：

| 位 | 值 | 含义 | 处理 |
|---|---|---|---|
| `USN_REASON_DATA_OVERWRITE/EXTEND/TRUNCATION` | 0x1/0x2/0x4 | 内容变化 | 需重新 stat |
| `USN_REASON_FILE_CREATE` | 0x100 | 新建 | 需 stat |
| `USN_REASON_FILE_DELETE` | 0x200 | 删除 | 进墓碑流程 |
| `USN_REASON_RENAME_OLD_NAME` | 0x1000 | 改名（旧） | 与 NEW 成对 |
| `USN_REASON_RENAME_NEW_NAME` | 0x2000 | 改名（新） | 产出 `Move(FRN)` |
| `USN_REASON_BASIC_INFO_CHANGE` | 0x8000 | 属性/时间戳变化 | 需 stat |
| `USN_REASON_CLOSE` | 0x80000000 | 句柄关闭 | 可作"写完了"的强信号 |

### 4.2 读取循环（每 1–2 秒一次，成本几乎为 0）

```csharp
using var vol = File.OpenHandle(@"\\.\C:", FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
// File.OpenHandle 拿不到卷句柄 → 实际用 CreateFileW(@"\\.\C:", GENERIC_READ, FILE_SHARE_READ|WRITE, ...)

var input = new READ_USN_JOURNAL_DATA_V0 {
    StartUsn = checkpoint.NextUsn,
    ReasonMask = 0xFFFFFFFF,
    ReturnOnlyOnClose = 0,
};
var outBuf = ArrayPool<byte>.Shared.Rent(64 * 1024);     // 增量读，64KB 足够
try {
    while (DeviceIoControl(vol, FSCTL_READ_USN_JOURNAL,
                           inBuf, inBufLen, outBuf, outBuf.Length, out int ret, IntPtr.Zero)) {
        var nextUsn = BitConverter.ToInt64(outBuf, 0);           // 输出前 8 字节 = 下次起点
        for (int off = 8; off < ret; off += (int)rec.RecordLength) {
            var rec = MemoryMarshal.Read<USN_RECORD_V2_HEADER>(outBuf.AsSpan(off));
            var name = Encoding.Unicode.GetString(outBuf, off + rec.FileNameOffset, rec.FileNameLength);
            Handle(rec, name);                                    // 只做"登记"，不在这里做 IO
        }
        checkpoint.NextUsn = nextUsn;                            // ★ 但必须先持久化 DirtyTodo 才能推进（见下方要点）
        if (ret <= 8) break;                                     // 没有新记录
    }
} finally { ArrayPool<byte>.Shared.Return(outBuf); }
```

要点：
1. **顺序铁律**：本批记录先展开为脏目录待办并写入 journal（`DirtyTodo`，fsync），**再**推进 `nextUsn`。
   反之（先推 USN 后处理）一旦崩溃，这段时间的变更**永久丢失**；按此顺序崩溃最多重复处理一批（幂等，无害）。
2. `ret <= 8` 表示无新记录；正常一轮往往只有 0 条，**CPU 与网络成本为 0**。
3. 每次读到 `FirstUsn` 之后的内容才算连续；否则失效（§4.4）。

### 4.3 路径解析：FRN → 路径

USN 只给 `FileReferenceNumber` + `ParentFileReferenceNumber`，**不给完整路径**。两种解法：

| 做法 | 成本 | 说明 |
|---|---|---|
| ① `OpenFileById(volumeHandle, FILE_ID_DESCRIPTOR{ FileId = frn })` + `GetFinalPathNameByHandle` | 每路径 1–2 syscall（本地，微秒级） | 简单、准确；NTFS 卷支持。**首选** |
| ② 维护 FRN→(parentFRN, name) 表（内存或 SQLite），向上拼路径 | 需要全量建表（O(N) 磁盘/内存） | 用于"改名"场景，避免逐个 OpenFileById |

推荐组合：**改名用 ②（只需处理成对的 OLD/NEW 记录），其他变化用 ①**。

```csharp
// 改名：RENAME_OLD_NAME 与 RENAME_NEW_NAME 同 FRN，按 USN 顺序配对
if ((rec.Reason & USN_REASON_RENAME_OLD_NAME) != 0) _pendingRename[rec.FileReferenceNumber] = GetPath(rec);
if ((rec.Reason & USN_REASON_RENAME_NEW_NAME) != 0 && _pendingRename.Remove(rec.FileReferenceNumber, out var old))
    ReportMove(old, GetPath(rec));
```

### 4.4 检查点表与失效检测（必须做，否则会静默漏同步）

```sql
CREATE TABLE usn_checkpoint (
  volume_id    TEXT PRIMARY KEY,   -- 卷序列号 + 挂载点，如 "C: 0x1a2b3c4d"
  journal_id   INTEGER NOT NULL,   -- UsnJournalID
  next_usn     INTEGER NOT NULL,   -- 下一条待读
  updated_at   INTEGER NOT NULL
);
```

失效判定（任一成立 → 作废检查点 + 全量重建，日志 `usn-invalidated`）：

| 条件 | 原因 |
|---|---|
| `UsnJournalID` 与库中不同 | 期刊被删除/重建（`fsutil usn deletejournal`、chkdsk、卷格式化） |
| `FirstUsn > next_usn` | 环形缓冲已覆盖（两次轮询间隔内写入 > 32MB） |
| 卷序列号变化 / 挂载点变化 | U 盘、VHD 挂载、跨机器复制 |
| 卷句柄打开失败（提权丢失） | 服务账户变更 |

### 4.5 与 v1 组件的协同（不用改的地方）

- **仍需 stat**：USN 只说"变了"，`size/mtime` 要重新取（**只对变更集取**，这是省 IO 的关键）。
- **仍需基线判等**：USN 也会记录**同步器自己写入**的文件 → `self-write` 抑制照旧；即使漏判，`L==B` 也会判 `None`，不会产生传输。
- **仍需远端兜底枚举**：USN 管不到 NAS。
- **USN 不提供 hashes**：指纹漏斗不变，只是"输入范围"变小。

### 4.6 部署形态与降级（feature flag）

```jsonc
"monitoring": {
  "mode": "auto",           // auto | usn | fsw
  "usnPollMs": 1000,        // 轮询间隔（决定丢事件风险 vs 开销）
  "fallbackFullScanMinutes": 30   // USN 生效时，全量校验频率可放宽
}
```

| 启动探测顺序 | 结果 |
|---|---|
| 非管理员 → `fsctl` 失败 | `monitor-mode=fsw`（日志 Warn：`usn-unavailable reason=privilege`） |
| 本地根不在 NTFS 卷 | `monitor-mode=fsw` |
| 期刊不存在（该卷 USN 关闭） | 尝试 `FSCTL_CREATE_USN_JOURNAL`（需提权，`MaximumSize=64MB`）；失败 → fsw |
| 检查点失效 | 全量重建一次，之后恢复 usn |
| 一切正常 | `monitor-mode=usn`，本地周期全量扫描可放宽到 30min 或关闭 |

> 这意味着：**不想提权也能用**，只是退回 v1 的 FSW 模式，功能不减、只是兜底扫描更频繁。

---

## 5. 内存压缩 v2（收益最大的一步，与 USN 无关）

### 5.1 手段与收益

| # | 措施 | 现在（v1） | 改法（v2） | 收益 |
|---|---|---|---|---|
| M1 | **取消内存清单**：扫描边枚举边写 `staging` 表，diff 用 SQL 做 join | `Dictionary<rel,FileEntry>` 10万条 ≈ 15MB，且随文件数线性增长 | `CREATE TEMP TABLE s_local(...) WITHOUT ROWID`，逐条 INSERT 到同一事务 | **−15MB**，内存变 **O(1)** |
| M2 | **路径模型**：`parent_id + name`（照抄 USN 的 FRN 思路），不存完整路径字符串 | 每条路径平均 60–90 字符 ≈ 120–180B | 每条 ≈ 40–50B + 一次 `dir_id` 复用 | **−60~100B/条** |
| M3 | **哈希用 BLOB**：快速指纹 `XxHash64`(8B)，全量 `SHA-256`(32B，仅必要文件) | 64 字符 hex 字符串 ≈ 128B（UTF-16）+ 分配 | `BLOB(8)` / `BLOB(32)` | **−100B/条** |
| M4 | **结构化 `struct` / `Span`**，路径用 `ReadOnlySpan<char>` + `ArrayPool<char>` | 每个文件 1 个 class + 2 个 string ≈ 200B 垃圾 | 零分配（栈/池） | GC 压力 −80% |
| M5 | **目录级批事务**：每处理完一个目录提交一次 | 全量后统一提交 | 常数内存，断点更细 | 峰值 −数 MB |
| M6 | **并发默认 1**，共享 1 个 256KB 缓冲 | 2 并发 × 128KB + 2 哈希上下文 | 1 缓冲 + 1 上下文 | −0.5MB |
| M7 | **GC 配置**：`ConcurrentGC=false`、`ConserveMemory=7`、`HeapHardLimit=96MB` | 并发 GC 后台线程 + 堆不主动收缩 | runtimeconfig 硬限 + 保守回收 | −5~10MB，且**防止缓慢漂移** |
| M8 | **SQLite 参数**：`cache_size=-4000`、`temp_store=FILE`、`mmap_size=0`、`journal=WAL` | 默认 2MB cache + mmap | 4MB 显式上限，临时表落盘 | 可控 |
| M9 | **事件队列**：定长 ring 8k + 溢出置 `NeedFullRescan` | 字典 5万条 + 10k 队列 | 定长环形缓冲 | −2~8MB |
| M10 | **日志**：SQLite 批量写 + 文件 sink `buffered: true, flushToDiskInterval: 5s` | 每行同步 flush + 字符串格式化 | 批量 | −1~3MB |

### 5.2 SQL 归并（M1 的具体做法：SQL 做集合运算，C# 做决策）

```sql
-- ① 扫描边枚举边写（流式，内存恒定）
CREATE TEMP TABLE IF NOT EXISTS s_local (rel TEXT PRIMARY KEY, size INTEGER, mtime INTEGER, hash BLOB) WITHOUT ROWID;
CREATE TEMP TABLE IF NOT EXISTS s_remote(rel TEXT PRIMARY KEY, size INTEGER, mtime INTEGER, hash BLOB) WITHOUT ROWID;
INSERT INTO s_local VALUES (?,?,?,?);        -- 复用 prepared statement，每目录事务提交

-- ② 一次 join 产出"事实流"（SQLite 3.39+ 支持 FULL OUTER JOIN）
CREATE TEMP TABLE IF NOT EXISTS facts AS
SELECT k.rel                      AS rel,
       l.size AS l_size, l.mtime  AS l_mtime,
       r.size AS r_size, r.mtime  AS r_mtime,
       b.size AS b_size, b.local_mtime AS b_local_mtime, b.remote_mtime AS b_remote_mtime
FROM (SELECT rel FROM s_local UNION SELECT rel FROM s_remote) k
LEFT JOIN s_local     l ON l.rel = k.rel
LEFT JOIN s_remote    r ON r.rel = k.rel
LEFT JOIN baseline    b ON b.rel = k.rel;   -- baseline = journal 的基线视图（或 sync_record 若用 sqlite provider）
```

然后 **C# 侧流式读 `facts`，逐行调用纯函数 `DiffEngine.Decide(...)`**（12 分支逻辑保持可单测，不进 SQL）：

```csharp
await foreach (var f in factsReader.ReadAllAsync(ct))   // 一次只驻留一行
    yield return DiffEngine.Decide(f.ToLocalEntry(), f.ToRemoteEntry(), f.ToBase());
```

> 若 SQLite 版本 < 3.39（不支持 FULL OUTER JOIN），用 `LEFT JOIN ... UNION ALL LEFT JOIN ...` 等价改写即可，已在测试里覆盖。

**为什么不在 SQL 里写 CASE 分支**：12 分支决策是最需要回归保护的地方，放在 C# 里可以对着 v1 的实现做**逐行等价测试**（同一输入序列，两种实现产出的计划必须完全相同）。这也是 §7 的第一条测试。

### 5.3 新版内存预算（10 万文件、1GB 单文件复制）

```
稳态（USN 轮询 + 每 30min 校验）                 峰值（复制 1GB，并发 1）
┌──────────────────────────────┐                ┌──────────────────────────────┐
│ 运行时/GC 自身          ~18M │                │ 运行时/GC 自身          ~18M │
│ SQLite page cache        4M │                │ SQLite page cache        4M │
│ 事件 ring(8k 定长)      ~1M │                │ 事件 ring               ~1M │
│ SQL staging 连接/语句   ~2M │                │ 复制缓冲 1×256KB      0.25M │
│ 余量                  ~10M │                │ 哈希上下文              ~1M │
│ ───────────────────────────  │                │ ───────────────────────────  │
│                       ~35M │                │                       ~24M │
└──────────────────────────────┘                └──────────────────────────────┘
硬限：GCHeapHardLimit=96MB（配置项，超限抛 OOM 而非拖死系统）
目标：稳态 < 40MB，峰值 < 50MB，且**与文件数、单文件大小无关**
```

对比 v1 的 **~70MB 稳态 / 与会话规模线性相关**，这是一次"结构性"压缩，不是抠参数。

### 5.4 内存自检守卫（最后一道保险）

```csharp
// 每 60s 采样；连续 3 次超阈值才动作，避免抖动
if (proc.PrivateMemorySize64 > _softLimitBytes) {   // 默认 softLimit=64MB
    GC.Collect(2, GCCollectionMode.Forced, blocking: true, compacting: true);
    SetProcessWorkingSetSize(proc.Handle, -1, -1);  // EmptyWorkingSet 等价的温和版本
    _log.Warn("memory-guard trimmed workingSet={Ws}MB", ws >> 20);
}
```
注意：`SetProcessWorkingSetSize(-1,-1)` 会把页面换出、下次访问有缺页开销，**只作为兜底**，并必须记日志（否则会掩盖真实泄漏）。

---

## 6. 对现有文档的增量变更

| 文档 | 变更 |
|---|---|
| `PRD.md` | 新增 **FR-2.7**（USN 增量监控 + 自动降级）、**FR-2.8**（staging + SQL 归并）、**FR-4.7**（监控模式自检与上报）；**NFR-1 收紧**为稳态 <40MB / 峰值 <50MB 且与规模无关；新增 **Q6**（是否允许提权运行以启用 USN）；新增配置段 `monitoring` |
| `技术方案.md` | §9 内存章节替换为本文 §5；§7.1 监听器章节增加"USN 优先、FSW 降级"；表结构新增 `usn_checkpoint`、`s_local/s_remote/facts`（TEMP） |
| `骨架图.md` | 新增"监控分层图"与"USN 轮询 → 变更集 → staging → 决策"数据流；线程骨架中本地 Watcher 标注 `USN | FSW` 二选一 |

---

## 7. 回归测试（v2 必须新增）

| # | 用例 | 判据 |
|---|---|---|
| T1 | **等价性**：同一操作序列（建/改/删/改名/冲突）下，`Decide()` 的 v1 内存实现与 v2 SQL 归并实现产出的计划**逐条相同** | diff 为空 |
| T2 | USN 模式与 FSW 模式跑同一操作序列 | 计划与执行结果相同 |
| T3 | 期刊失效（`fsutil usn deletejournal /d C:` 后重建） | 自动全量重建，日志 `usn-invalidated`，无漏同步 |
| T4 | 改名 1000 个文件 | 计划含 1000 个 `Move`，`bytes=0` |
| T5 | 非管理员启动 | `monitor-mode=fsw`，功能完整，日志明确 |
| T6 | 内存压测：10 万文件索引 + 1GB 复制 | 峰值 < 50MB；`dotnet-counters` 采样无锯齿上涨 |
| T7 | USN 轮询期间产生 > 32MB 变更（大批量解压） | 检测到 `FirstUsn > checkpoint` → 降级全量，不丢文件 |
| T8 | 断网 30 分钟后恢复 | 远端 5min 内补齐；本地 USN 断点续读正确 |

---

## 8. 实施顺序（每步独立可回退）

| 步骤 | 内容 | 风险 | 回退方式 |
|---|---|---|---|
| S1 | 内存压缩 M1（staging + SQL 归并） | 低 | 保留 v1 `Dictionary` 路径，配置 `diffMode=memory|sql` |
| S2 | M2/M3/M4（parent_id 模型 + BLOB 哈希 + 零分配扫描） | 中（要迁移表结构） | 迁移脚本加版本号，可回滚到 v1 schema |
| S3 | M7/M8/M9/M10（GC/SQLite/队列/日志参数） | 低 | 纯配置 |
| S4 | USN 监控（`monitoring.mode=usn`） | 中（需提权 + 期刊失效处理） | `mode=fsw` 一键回退 |
| S5 | （可选）`FSCTL_ENUM_USN_DATA` 首轮 MFT 索引 | 低 | 默认关闭 |

**建议**：S1–S3 先做（不依赖权限、收益确定、可测）；S4 作为第二阶段，由 Q6 的答复决定是否启用。

---

## 9. 风险与对策（v2 新增部分）

| 风险 | 影响 | 对策 |
|---|---|---|
| 需要提权，用户不接受 | USN 不可用 | `mode=auto` 自动降级到 FSW；功能不减，只牺牲"零丢事件 + 扫描成本" |
| 两次轮询间变更超 32MB 覆盖 | 漏同步 | 1–2s 轮询 + `FirstUsn > checkpoint` 失效检测 + 全量重建 |
| FRN 复用到不同文件（旧文件删后新文件复用同 FRN） | 误判改名 | 改名配对时校验 `TimeStamp` 接近且 `FileAttributes` 一致；否则按"删+建"处理 |
| USN 记录含 ADS/临时流 | 噪声 | 过滤 `FileAttributes` 与流名（含 `:` 的名字段） |
| staging 表写放大、临时文件占盘 | 磁盘压力 | `TEMP` 表（`temp_store=FILE` 落在 `%TEMP%`），每轮 `DROP`；限制单轮条目数，超限分批 |
| 决策逻辑改成 SQL 后难测 | 回归风险 | **决策留在 C# 纯函数**，SQL 只做集合运算；T1 等价性测试强制 |
| 内存硬限 96MB 触发 OOM | 进程退出 | 硬限作为兜底而非目标；压测保证峰值 < 50MB；OOM 时保留状态库不影响正确性 |
| `SetProcessWorkingSetSize` 掩盖泄漏 | 掩盖问题 | 每次 trim 必须记日志 + 指标上报 |

---

## 10. 一页总结

- **USN 值不值得做**：值得，但只用在本机 NTFS 侧；它把"本地扫描"从 O(全树) 变成 O(变化数) 并消灭丢事件风险，代价是必须提权 + 处理期刊失效。远端（NAS/SMB）永远只能 `ReadDirectoryChangesW` + 周期枚举。
- **内存优化的真正杀招**不是 USN，而是 **M1（清单搬进 SQLite，SQL 做 join，C# 只做决策）**：一次拿到 −15MB 且把内存从"随文件数线性"变成"常数"。
- **最终指标**：稳态 < 40MB、峰值 < 50MB、与文件数/文件大小无关；USN 模式下 CPU 空闲占用接近 0。

---

## 11. 实测：每种记录模型的单条成本（.NET 8 / 本机）

> 基准程序：`/tmp/memprobe`（.NET 8 控制台 + Microsoft.Data.Sqlite 8.0.13），逐个模型装载 N 条记录后 `GC.Collect(2, Forced, compacting)` 再取 `GC.GetTotalMemory` 与 `Process.PrivateMemorySize64`。
> 路径样例：`docs\部门{n%100}\2024年度\项目文档-{n}\季度报表-{n}.xlsx`（≈62 字符）。

| 模型 | 10,000 文件 | 100,000 文件 | 300,000 文件 | 每条 |
|---|---|---|---|---|
| 空 .NET 8 控制台（地板） | managed 0.3MB / **WS 27.8MB** | 同 | 同 | — |
| **A. v1：`Dictionary<string, class>` 全路径** | 2.2MB / WS 36.6 | 19.6MB / WS 60.1 | 54.7MB / WS 107.3 | **206–232 B**，线性 |
| B. `Dictionary<string, struct>` 全路径 | 2.1MB | 18.7MB | 48.3MB | 191 B（省得有限，字符串才是大头） |
| **C. 名字 arena + parent_id（USN FRN 模型）** | 1.1MB | 8.7MB | 25.5MB | **69 B** |
| **D. SQLite TEMP staging（temp_store=FILE）** | **0.3MB** | **0.3MB** | **0.3MB** | **0 B（托管堆）**，行落盘 |
| E. merge-walk：只驻留 1 个目录（1000 条） | 0.3MB | 0.3MB | 0.3MB | 与总文件数**无关** |
| F. + 256KB 复制缓冲 | +0.3MB | +0.3MB | +0.3MB | 与单文件大小无关 |

附带性能数据（同一基准）：staging 插入 **1 万行 33ms / 10 万行 291ms / 30 万行 720ms**（单事务、prepared statement）；流式全表扫描 10 万行托管堆仍为 **0.3MB**。

**两个可复用的结论**

1. **压缩"每条记录"的收益是线性的，压缩"是否全量驻留"的收益是结构性的**：C 相对 A 只省 3 倍，D/E 直接省掉整个维度。
2. **Windows 工作集不会自动回落**（实测：清空集合后 WS 仍停留高位，只有 SQLite 释放原生内存后才降）。所以"内存曲线看起来不降"不一定是泄漏，但也说明 **大对象阶段结束后应主动 trim 一次并记日志**（§5.4）。

---

## 12. 两个指定文件夹的最终形态：目录级合并遍历（merge-walk）

需求是"本地文件夹 A ⇄ NAS 文件夹 B"，**不是全盘**，因此完全不需要"两侧各建一份全量清单"：

```
同步(A, B):
  for each 目录 d（A 与 B 同名并行下钻）:
      读 A\d 的条目（流式，FindFirstFileEx 批量）
      读 B\d 的条目
      本地 diff 这两个列表 → 动作（内存 = 该目录条目数 × ~70B）
      提交 → 递归子目录
```

- 内存 = **最深路径 + 最宽目录**，与总文件数无关；典型办公目录 200–2000 个条目 → **<1MB**。
- 需要跨目录改名识别时，只保留一个**有上限的候选索引**（`(size,quickHash) → 路径`，默认上限 20,000 条，超限降级为"复制而非改名"），而不是全量哈希表。
- 跨轮次状态（基线/日志/USN 检查点）本来就该在 SQLite 里，不进内存。
- 目录级快速跳过（两侧目录 mtime + 条目数与基线一致 → 跳过下钻）**只能省 IO，不能替代内容校验**：文件内容原地修改不会改父目录 mtime，必须靠事件（USN/FSW）或周期全量校验兜底，否则会漏同步。

因此最终内存构成（100k 文件、最大目录 2k 条）：

```
.NET 运行时地板        27.8MB   ← 实测，压不下去
merge-walk 目录条目    ~0.15MB  ← 2000 × 70B
SQLite 页缓存          4.0MB   ← PRAGMA cache_size=-4000
复制缓冲(256KB×1)      0.25MB
日志/事件 ring         ~1.0MB
──────────────────────────────
稳态                  ~33MB   （与文件总数无关）
峰值（1GB 复制）       ~34MB
```


---

## v2.0 统一修订（2026-11-23）

低内存目标继续作为 P0，不得通过升级 UI 框架、全量文件清单或无界队列牺牲。内存指标统一参考 产品定义与开发基线-v2.md 第 4 节：稳态目标 <40MB、普通同步峰值目标 <50MB、UI 独立进程目标 4–7MB，并要求注明测试环境与测量口径。状态记录允许随文件量增长，但运行时内存通过磁盘持久化、分页/有界缓存和流式遍历控制；不得继续使用“与文件数量绝对无关”这种不严谨表述。
