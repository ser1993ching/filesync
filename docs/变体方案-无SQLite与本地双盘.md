# 变体方案｜① 不用 SQLite ② 任意两路径对同步（含本地 C↔D）

> 配套：`docs/PRD.md`、`docs/骨架图.md`、`docs/优化方案-USN与内存.md`。
> 本文回答三个问题：**能不能不要 SQLite？本地 C:\A ⇄ D:\B 这类同机双目录实时同步包不包含？怎么把"NAS 映射盘"与"本地双盘"统一到同一套引擎里？**

---

## 0. 结论

| 问题 | 回答 |
|---|---|
| 能不能不用 SQLite？ | **能，而且对"两个文件夹同步"这个规模，不用 SQLite 更轻**：引擎只需要三件事——① 按路径点查基线 ② 追加日志 ③ 原子更新。这三件事用"追加日志 + 快照 + CRC"的纯托管实现即可，单文件 exe 不引入 native 依赖，体积少 1.5MB 左右。 |
| 包不包含本地 C:\A ⇄ D:\B？ | **包含，而且是同一套引擎，只是"能力档位"不同**。NAS 映射盘不是特例，它只是"另一方是 SMB 路径"的一个 pair。 |
| 为什么可以统一？ | 引擎只通过 Win32 文件 API 访问两侧根目录；差异（有没有 USN、事件是否可靠、mtime 精度、改名是否原子）全部收敛到**每侧一份能力档案（capability profile）**里，不影响判定逻辑。 |
| 本地双盘有什么额外坑？ | ① 跨卷 `File.Move` 不是 rename 而是**复制**（实测 64MB：同卷 1.5ms，跨卷 26ms ≈ 2.4GB/s）；② 两根本同卷且互相包含 → 无限回环，必须拒绝；③ junction/symlink 可能把 A 指回 B。 |

---

## 1. 状态后端：三种无 SQLite 实现 + 对比

引擎对状态层的接口只有这几个方法（`IStateStore`）：

```csharp
interface IStateStore : IDisposable
{
    bool TryGet(string relPath, out Baseline b);                 // 点查基线（热路径，每文件 1 次）
    void Upd(Baseline b, LogEntry log);                          // 原子：基线 + 日志一起生效
    void Delete(string relPath, LogEntry log);
    IEnumerable<LogEntry> QueryLog(LogQuery q);                  // 日志检索
    void SaveInflight(string relPath, Direction d, string tmp);
    void ClearInflight(string relPath);
    bool TryGetUsnCheckpoint(string volId, out UsnCheckpoint c);  // USN 续读点
    void SaveUsnCheckpoint(UsnCheckpoint c);
}
```

### 方案 A：追加日志 + 周期快照（推荐默认，纯托管）

```
data/
├─ state.snapshot      # 全量基线快照（二进制定长记录 + 字符串 arena），每次压实重写
├─ state.journal       # 追加式增量记录：Upd/Del/Inflight/Checkpoint，每条带 CRC32
└─ logs/fs-YYYYMMDD.jsonl   # 人读/可 grep 的结构化日志（滚动）
```

- 记录格式：`[u32 len][u32 crc32][payload]`，payload 用 `BinaryWriter` 写相对路径、size、mtime、hash、op、ts。
- **崩溃安全**：重放时遇到 CRC 不符或长度越界的尾部记录 → 截断到上一条有效记录（经典 WAL tail 处理），不需要事务引擎。
- **启动成本**：读快照 + 重放尾日志。日志 > 快照 2 倍时压实（写新快照 + 原子 rename + 清空日志）。
- **内存**：`Dictionary<string, Baseline>`，用 arena + parent_id 时 **69B/文件**（实测），10 万文件 ≈ 7MB；若要严格 O(1)，换成下面的"磁盘哈希索引"。
- **磁盘哈希索引（可选，严格 O(1)）**：`baseline.idx` 是定长 32–64B 槽的开放寻址表，key = `XxHash64(relPath)`，value = (arena 偏移, 长度, size, mtime, hash)；命中只需 1–2 次 `pread`。写时双写（idx + arena）并用 seq 号做一致性校验。
- 日志：JSONL 直接 `grep`/`tail`，`logs` 子命令在内存里流式过滤（不整文件加载）。

### 方案 B：每目录清单（manifest），天然 O(1)

```
A\.filesync\manifests\<dirRelHash>.jsonl     # 该目录上次同步的条目：name/size/mtime/hash
```

- merge-walk 到某个目录时，只读写**这个目录的清单文件**（几百条），内存与总规模无关；不需要全局索引，也不需要启动重建。
- 删除判定：清单里有、本侧没有 → 新删；两侧都没有 → 清清单条目。
- 代价：① 目录数 × 1 个小文件（10 万目录 = 10 万个小文件，在 SMB 上是灾难，**仅适合本地双盘**）；② 跨目录改名识别退化为"复制"；③ 全局查询（"哪些文件最近同步过"）需要扫所有清单。
- 适合：本地 C↔D、目录数不多、且希望"零外部依赖 + 零启动成本"的场景。

### 方案 C：SQLite（现有方案，保留为可选）

优点是事务、索引、可查询日志、启动 O(1)；代价是多 ~1.3MB native 依赖、WAL 文件、`.db` 不能放 NAS 共享上、以及"锁/损坏"这类运维面。**作为 provider 保留，不作为唯一实现。**

### 对比

| 维度 | A. journal+snapshot | B. 每目录 manifest | C. SQLite |
|---|---|---|---|
| 托管内存 | 69B/文件（或 O(1)，用磁盘索引） | **O(1)** | O(1) |
| 启动成本 | 重放尾日志（10 万条 ≈ 0.3–1s） | **0** | ~10ms |
| 崩溃安全 | 好（CRC + 截断尾部） | 好（tmp+rename 原子替换） | 好（事务） |
| 日志可查询 | 好（JSONL / `logs` 子命令） | 一般（分散） | **最好（SQL）** |
| 依赖 / exe 体积 | **纯托管，+0 依赖** | **纯托管，+0** | +1.3MB native |
| 放在 NAS 侧 | 可以（只写本地） | 不建议（小文件多） | 禁止 |
| 实现工作量 | 中（~400 行） | 低（~200 行） | 低（现成） |
| 跨目录改名识别 | 可以 | 退化为复制 | 可以 |

**建议**：`state.provider = journal`（默认）｜`manifest`（本地双盘、目录数可控）｜`sqlite`（想要 SQL 查日志时）。三者共用同一套 `Decide()` 判定与 `Executor`，用"同一操作序列、三种 provider 产出计划逐条相同"的等价性测试保护（现有 T1 测试扩展即可）。

---

## 2. 统一模型：pair + 能力档案

```csharp
sealed record RootProfile(
    string Path,                      // C:\A / D:\B / Z:\Docs / \\NAS\share\Docs
    RootKind Kind,                    // LocalFixed | LocalRemovable | Network | Junction
    FileSystemKind Fs,                // Ntfs | ReFs | ExFat | Fat32 | Unknown
    bool HasUsn,                      // 能否 FSCTL_READ_USN_JOURNAL（提权 + NTFS）
    bool WatcherReliable,             // 本地 NTFS=true；SMB=false（会丢事件）
    bool AtomicRename,                // 同卷=true；跨卷=false（Move 会退化成复制）
    int  MtimeToleranceSec,           // 本地 0；SMB/exFAT 1–2
    bool MtimePreservedAfterCopy,     // 跨卷 Move / SMB 复制后是否可靠
    bool SupportsFileId,              // 能否用 FileId 做 O(1) 改名识别
    bool CaseInsensitive);
```

探测手段（启动时一次，结果写入 `status` 与日志）：

| 字段 | 探测方式 |
|---|---|
| `Kind` | `GetDriveTypeW`（DRIVE_FIXED / DRIVE_REMOTE / DRIVE_REMOVABLE）、`WNetGetConnectionW` 反查 UNC |
| `Fs` | `GetVolumeInformationW` 取文件系统名（NTFS / ReFS / exFAT / FAT32） |
| `HasUsn` | 提权 + `FSCTL_QUERY_USN_JOURNAL` 成功 |
| `AtomicRename` | 两根本在同一卷（vol serial 相同）→ true |
| `MtimeToleranceSec` | 按 `Fs`/`Kind` 预设（NTFS 0；SMB 2；exFAT 2） |
| `SupportsFileId` | `GetFileInformationByHandleEx(FileIdInfo)` 成功（SMB 上时有时无） |

**判定逻辑（`Decide`）与执行顺序完全不变**——变的只是：
1. 监听通道（USN / FSW / 周期扫描，见 `优化方案` §13 分层图）；
2. `Move` 动作的**成本模型**（原子 rename vs 跨卷复制）；
3. mtime 比较的容差；
4. 兜底扫描的频率（本地 60min，SMB 5–10min）。

### 能力矩阵（同一引擎的三种典型 pair）

| Pair 类型 | 示例 | A 侧监控 | B 侧监控 | Move 成本 | 兜底扫描 |
|---|---|---|---|---|---|
| 本地 ⇄ 本地（同卷） | `C:\A` ⇄ `C:\B` | USN 或 FSW | USN 或 FSW | 原子 rename（1.5ms） | 双方 60min |
| 本地 ⇄ 本地（**跨卷**） | `C:\A` ⇄ `D:\B` | USN/FSW | USN/FSW | **B 侧 rename 仍原子**（都在 D: 内）；A↔B 内容搬运是复制 | 双方 60min |
| 本地 ⇄ NAS | `D:\Work` ⇄ `Z:\Docs` | USN/FSW | **只能 FSW**（丢事件） | B 侧同卷 rename（SMB 支持），但延迟高 | 本地 30–60min；远端 5–10min |

> 关键澄清：`C:\A ⇄ D:\B` 里"把 `C:\A\x.txt` 改名成 `C:\A\y.txt`"→ 需要在 B 侧执行改名，而 B 侧改名发生在 `D:` 卷内部，**是原子的、几乎零成本**。真正跨卷的复制只发生在"内容搬运"时，那是本就有成本的正常传输。"跨卷 Move 退化成复制"影响的是**极端情况**——例如把文件从 `C:\A` 移动到 `C:\A\sub\`（同卷，便宜），而把文件在 B 侧"重命名到另一个盘"这种事在 pair 内不会发生。

---

## 3. 本地 C:\A ⇄ D:\B 实时同步的执行方案（逐条落地）

### 3.1 配置

```jsonc
{
  "state": { "provider": "journal", "dir": "C:\\FileSync\\data" },
  "pairs": [
    {
      "name": "local-cd",
      "a": { "root": "C:\\A" },
      "b": { "root": "D:\\B" },
      "direction": "TwoWay",
      "conflictPolicy": "KeepBoth",
      "deletePolicy": "Recycle",
      "monitoring": { "mode": "auto", "debounceQuietMs": 2000 },
      "performance": { "maxParallelTransfers": 2, "bufferBytes": 262144, "hashMode": "Quick" }
    },
    {
      "name": "nas-docs",
      "a": { "root": "D:\\Work\\Docs" },
      "b": { "root": "\\\\NAS\\share\\Docs" },     // 或 Z:\Docs，启动时解析为 UNC
      "monitoring": { "mode": "auto", "debounceQuietMs": 3000 },
      "performance": { "maxParallelTransfers": 1 }
    }
  ]
}
```

### 3.2 启动校验（本地双盘特有，必须做）

| 校验 | 判据 | 失败处理 |
|---|---|---|
| 两根不相同 | 规范化后 `!=` | 退出码 5 |
| **不互相包含** | 一侧不能是另一侧的祖先目录 | 退出码 5（否则无限回环） |
| 同一物理目录 | `VolumeSerial + FileId` 相同（防 junction/挂载点） | 退出码 5 |
| 卷类型 | `DRIVE_FIXED/REMOVABLE/REMOTE` | 网络卷 → 走 SMB 档位；可移动卷 → mtime 容差 2s |
| 权限 | 两侧均可枚举 + 可写（小文件试写后删除） | 退出码 5 |
| **`data\` 不在任一 root 内** | 路径规范化后比较（含短名/符号链接） | 退出码 5（否则状态库会自己同步自己） |
| **跨 pair 根不重叠** | 所有 pair 的 A/B 根两两不成祖先关系 | 退出码 5，日志指明冲突的两个 pair |

### 3.3 监听与实时性

- **两侧都本地 NTFS 时**：`mode=auto` 会各开一个 USN 轮询（有权限）或 FSW（无权限）。本地双盘是 USN 收益最大的场景——两侧都能"零丢事件 + O(变化数)扫描"，实际接近"改完即同步"。
- 防抖/探测/锁文件（`~$`）逻辑完全复用 §PRD FR-4；本地 Office 编辑同样会加锁，探测同样有效。
- 兜底扫描：本地 60min 一次元数据校验（USN 生效时）或 10min（FSW 模式）。

### 3.4 执行细节差异（相对 NAS 方案）

| 环节 | 变化 |
|---|---|
| 传输 | 本地盘吞吐远高于 SMB，`maxParallelTransfers` 可放到 2–4（内存 +0.25MB/并发）；仍禁止把整文件读进内存 |
| Move | B 侧改名在 `D:` 内 → `File.Move` 原子；跨卷极端情况按"复制 + 删除"记日志并计入 bytes |
| 复制后校验 | **跨卷 Move 后必须重新 stat**（时间戳/属性可能不被保留），基线以实际值为准 |
| 删除 | 回收区放在**各自 pair 的数据目录**（`data/trash/<pair>/<ts>/`），不要放进 A/B 本身，避免被自己扫到 |
| 自写回环 | 与 NAS 相同：`self-write` 登记 + 基线判等双保险 |
| 磁盘满 | 本地满盘风险更真实：执行前检查目标卷剩余空间 > 文件大小 × 1.1，否则跳过并 `Error disk-full` |
| 断电 | 每目录/每文件事务提交；启动清 `.filesync-tmp`；`inflight` 重传 |

### 3.5 性能预期（本地双盘，无额外参数）

| 指标 | 预期 |
|---|---|
| 事件 → 同步完成（小文件） | 1–3s（debounce 2s + 探测 1s） |
| 单大文件吞吐 | 受磁盘限制（NVMe 实测复制 64MB ≈ 26ms ≈ 2.4GB/s） |
| 1 万文件首次全量 | 元数据模式数十秒级 |
| 稳态内存 | ≈ 33MB（与文件数、文件大小无关，见 `优化方案` §12） |

---

## 4. 对 PRD 的增量

| 项 | 变更 |
|---|---|
| FR-1.5（新） | 支持 `pairs[]` 多对路径，每对独立的根、方向、策略、监听档位；一对失败不影响其他对 |
| FR-2.10（新） | 启动探测两侧能力档案（Kind/Fs/USN/AtomicRename/mtime 容差/FileId），写入 `status` 与日志；判定逻辑不随档位改变 |
| FR-1.6（新） | 启动校验：两根相同/互相包含/同物理目录（junction）/不可写 → 退出码 5 |
| §7.1 数据模型 | 状态后端改为可插拔：`journal`（默认）/`manifest`（本地双盘）/`sqlite`（可选）；`sync_record` 不再是唯一实现 |
| NFR-6 | 崩溃安全不依赖 SQLite 事务：journal 用 CRC + 尾部截断；manifest 用 tmp+rename |
| Q7（新） | 默认状态后端选 `journal`（无依赖）还是 `sqlite`（可 SQL 查日志）？ |

---

## 5. 测试增量

| # | 用例 | 判据 |
|---|---|---|
| V1 | `C:\A` ⇄ `D:\B` 双向：建/改/删/改名/目录移动 | 两端收敛，`bytes` 与 Move 判定正确 |
| V2 | 同卷 `C:\A` ⇄ `C:\B` 改名 1000 文件 | 1000 个 Move，`bytes=0`，耗时 < 2s |
| V3 | 跨卷内容复制 1GB | 吞吐达磁盘上限；复制后 stat 与基线一致 |
| V4 | 两根互相包含（`C:\A` 与 `C:\A\sub`） | 启动即拒绝，退出码 5 |
| V5 | B 是 A 的 junction | 通过 VolumeSerial+FileId 识别并拒绝 |
| V6 | 三种 `state.provider` 跑同一操作序列 | 产出的计划逐条相同（等价性） |
| V7 | 断电模拟（kill -9）三次 | 无损坏基线、无残留 tmp、正确重传 |
| V8 | 本地双盘 + NAS 两对同时运行 | 互不干扰，总内存仍 < 50MB |
