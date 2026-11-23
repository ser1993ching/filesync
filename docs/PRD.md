> **v2.0 总纲提示（2026-11-23）**：当前产品与实现以 [产品定义与开发基线 v2.0](产品定义与开发基线-v2.md) 为准。固定使用 .NET 8；低内存为 P0 核心优势；A/B 两端对等，默认双向，必须完整支持 NAS → 本地盘和本地盘 → NAS；无 SQLite，使用 journal + snapshot；NAS 不安装 Agent；NAS/SMB 通过 FSW 提示与周期对账监控。本文早期条目若与总纲冲突，以总纲为准，后续按总纲逐项清理。

---

# PRD｜本地磁盘 ⇄ NAS 映射盘 双向实时同步工具（FileSync）

| 项 | 内容 |
|---|---|
| 产品名（暂定） | FileSync（可执行文件名 `FileSync.Host.exe`） |
| 文档版本 | v1.0 |
| 状态 | v1.1 定稿（Q1–Q7 均已决策：USN 启用 / state=journal / diff=merge / UI=自绘独立进程） |
| 目标平台 | Windows 10 / 11 x64 |
| 技术栈约束 | .NET 8（本机 SDK 8.0.406，运行库 8.0.13）、**无 SQLite（默认 journal，纯托管）**、Serilog、单文件自包含发布 |
| 关联文档 | `docs/技术方案.md`（v1 实现细节）、`docs/骨架图.md`（架构与流程图）、`docs/优化方案-USN与内存.md`（v2 优化）、`docs/变体方案-无SQLite与本地双盘.md`（无 SQLite / 任意两路径对 / 本地双盘）、`docs/监控设计.md`（**监控定稿**）、`docs/UI方案-自绘低内存.md`（自绘 UI） |
| 交付形态 | CLI + Windows 服务（LocalSystem）+ 可选自绘 UI 进程（`FileSync.Ui.exe`）+ `data\` 状态/日志目录（journal，无 SQLite） |

---

## 1. 背景与问题

用户把局域网 NAS 共享目录映射为本地 `Z:` 盘，希望本地某个文件夹与 NAS 某个文件夹**双向同步**，并保留可审计的同步日志。

当前痛点（现状 = 手工操作）：

| 现状做法 | 问题 |
|---|---|
| 手工拖拽 / 复制粘贴 | 忘记同步、覆盖新版本、不知道哪边是最新 |
| `robocopy /MIR` 单向镜像 | 会把对面修改抹掉；无"谁改的"概念，两端都改必丢数据 |
| 云盘客户端（OneDrive 等） | 不支持任意 SMB/NAS 共享；或需要改造 NAS |
| 直接编辑 `Z:` 上的文件 | 断网/休眠时编辑丢内容；本地副本与远端不同步 |

**核心洞察**：双向同步的难点不是"复制文件"，而是三个判断——
① 谁改的（需要"上次同步基线"）；② 文件改完了没有（SMB 上编辑中的文件读到的可能是半成品）；③ 两端都改时怎么办（不能静默覆盖）。

## 2. 目标与非目标

### 2.1 v1 目标（必须达成）

| ID | 目标 | 度量 |
|---|---|---|
| G1 | 任一方向的改动 1 分钟内同步到对端（无占用的普通文件） | 事件→完成耗时 P95 < 8s，P99 < 60s |
| G2 | 正在编辑的文件不被同步，编辑结束后自动同步一次 | 同一文档连续保存 20 次 → 恰好 1 次传输，内容为最终版 |
| G3 | 两端同时修改不丢数据 | 100% 产生冲突副本，两份内容都可取回 |
| G4 | 每次同步可审计 | 100% 的传输/删除/冲突/跳过在 journal `LogEntry` 有记录，可按路径/时间/操作检索（`logs` 子命令） |
| G5 | 内存可控，不 OOM | 引擎稳态 < 40MB、复制 1GB 峰值 < 50MB，**与文件数无关**；可选 UI 进程 4–7MB（关闭即释放） |
| G6 | 断网/离线不造成数据损坏 | NAS 断连 30 分钟 → 0 个本地文件被删；恢复后自动补齐 |

### 2.2 非目标（v1 明确不做）

- ❌ 不做 NAS 端 agent / 不改造 NAS（只用已有的 SMB 映射）
- ❌ 不做块级增量传输（rsync/rdiff）——先靠"改名识别 + 元数据漏斗"降低流量
- ❌ 不做图形界面（v1 只有 CLI；托盘/GUI 列入 v2 候选）
- ❌ 不做多设备拓扑同步（只有"本地文件夹 ↔ 一个远端文件夹"这一对，多个 pair 串行执行）
- ❌ 不做版本历史服务（只提供删除/覆盖回收区，保留 N 天）
- ❌ 不做加密传输（依赖 SMB 自身加密/局域网信任模型）

## 3. 用户与使用场景

**主用户**：单一开发者/办公用户（即本机使用者），NAS 在局域网内，需长期无人值守运行。

| # | 场景 | 期望行为 |
|---|---|---|
| S1 | 在本地 `D:\Work\Docs` 新建/修改文件 | 数秒内出现在 `Z:\Docs`，日志记录 `Push` |
| S2 | 在 NAS 上（或另一台机器经 SMB）修改文件 | 数秒内拉回本地，日志记录 `Pull` |
| S3 | 在 NAS 上用 Word 打开并修改 `报表.docx`，60 秒后关闭 | Word 打开期间**不同步**，关闭后自动同步最终版本；期间日志有 `Skip locked-by-office` |
| S4 | 本地与远端各自改了同一个文件 | 两端都在，本地文件名加 `.conflict-<时间>-Local`，日志 `Conflict`，不丢任何一份 |
| S5 | 在 NAS 上误删了一片目录 | 本地不立即删除：默认进 `trash\`，日志 `DeleteLocal (recycled)`；单轮删除超阈值直接中止并告警 |
| S6 | NAS 掉线 / Z: 未映射 / 电脑休眠唤醒 | 停止同步、日志 `remote-unavailable`、**不删除任何文件**，恢复后自愈 |
| S7 | 想先看看会发生什么 | `sync --dry-run` 只打印计划，不动任何文件 |
| S8 | 事后追查"上周三那个文件到底同步了没" | `logs --path "docs/*" --since 2025-01-01` 精确回答 |

## 4. 术语（全文统一）

| 术语 | 定义 |
|---|---|
| **基线 / base** | 上一次成功同步完成时，双方内容一致的快照（size + mtime + 可选 hash），存在 `sync_record` 表 |
| **指纹** | 判定"文件是否变化"的证据：size / mtime / 快哈希 / 全哈希 |
| **冲突** | 基线之后，本地与远端都发生了变化且内容不同 |
| **墓碑** | 一端删除后，在状态库中标记 `deleted` 等待另一端确认的记录 |
| **self-write** | 同步器自己写入目标端产生的文件事件，需被忽略以防回环 |
| **对冲 / dry-run** | 只生成计划不执行 |
| **锁文件** | Office/WPS 编辑时产生的 `~$name.ext`、`.~lock.name#` |

## 5. 功能需求

优先级：P0 = v1 必须；P1 = v1 应有；P2 = 后续。

### FR-1 配置管理（P0）

| ID | 需求 | 验收标准 |
|---|---|---|
| FR-1.1 | 单一配置文件 `filesync.json`，支持 `localRoot` / `remoteRoot`（盘符或 UNC） | 两种写法都能启动；`remoteRoot` 为盘符时启动日志打印解析后的 UNC |
| FR-1.2 | 启动时校验：路径存在、两侧不同、非包含关系、可读写 | 校验失败退出码 5，错误信息指明具体项 |
| FR-1.3 | 配置缺失字段使用内置默认值 | 给最小配置（仅两个 root）可正常运行 |
| FR-1.4 | 配置项变更需重启生效，`status` 显示当前生效配置的关键项 | `status` 输出含 localRoot/remoteRoot/策略/阈值 |
| FR-1.5 | **多对路径**：`pairs[]` 支持任意两路径对（本地↔本地 / 本地↔NAS / 同卷 / 跨卷），每对独立方向/策略/监听档位；一对失败不影响其他对 | 同时跑 `C:\A⇄D:\B` 与 `本地⇄NAS` 两对，互不干扰，总内存仍 < 50MB |
| FR-1.6 | 启动校验：两根相同 / 互相包含 / 同一物理目录（junction） / 不可写 / **`data\` 落在 root 内** / **跨 pair 的根重叠** → 拒绝启动（退出码 5） | `C:\A` 与 `C:\A\sub` 直接拒绝；`C:\A⇄D:\B` 与 `C:\A\sub⇄D:\C` 同时存在时拒绝并指明冲突 pair |
| FR-1.7 | **首轮识别（只读清点）**：某 pair 首次建立同步时，先扫描两侧并分为 6 类（相同 / 仅 A / 仅 B / 两侧内容不同 / 仅元数据不同 / 已排除），输出摘要 + 可选明细报告（CSV/MD）；**用户拍板零写入**（不建基线、不动文件） | 首跑后两侧文件 hash 与时间戳完全未变；摘要计数与实际一致；报告可离线审阅 |
| FR-1.8 | **首轮决策五选一**：`KeepBoth` / `PreferA` / `PreferB` / `NewOnly`（只搬单侧独有项，差异项挂起） / `Skip`（只出报告，不建立同步）；支持 `--initial=` 无交互指定；无 TTY 且未指定 → 退出码 8 | 交互与 `--initial=prefer-a` 行为一致；`Skip` 后重启会重新询问；选择结果与清单指纹一起记入 journal |
| FR-1.9 | **破坏性语义必须显式确认**：`PreferX` 隐含“对方独有文件下架”，默认进 `data\trash\` 而非直删，需 `--confirm` 才执行，且可先 `--dry-run` 预览逐条计划 | 未加 `--confirm` 时只输出计划 + **退出码 9**；加后执行且 trash 可找回 |

### FR-2 扫描与指纹（P0）

| ID | 需求 | 验收标准 |
|---|---|---|
| FR-2.1 | 流式枚举两侧目录树（不构造全量 List），跳过排除规则匹配项 | 10 万文件扫描内存峰值 < 50MB（`diff.mode=merge` 下游标 ≈0.15MB） |
| FR-2.2 | 三级指纹漏斗：size → (size+mtime±2s) → 前后各 64KB 快哈希 → 全量 SHA-256 | 大文件重复扫描不重读全文件（`--stats` 可见 `bytesRead` 增量近 0） |
| FR-2.3 | 忽略规则支持 glob（`**/~$*`、`**/*.tmp`、`**/.git/**`） | 命中项不出现在计划中；`--verbose` 打印 `ignored` 计数 |
| FR-2.11 | **排除规则（可配置，三层）**：① 全局默认（Office 锁文件 `~$*`、`*.tmp`、`*.crdownload`、`Thumbs.db`、`.filesync-*`）② pair 级 `exclude[]` ③ 目录级 `.filesyncignore`（放在目录里，作用于该子树）。支持 glob 与目录子树排除；**排除项只进不进**（不扫不传不删）；命中目录直接剪枝不枚举 | `filesync rules test "C:\A\~$报表.docx"` 返回 `excluded by default:~$*`；排除目录在扫描日志中被剪枝（`pruned=1`） |
| FR-2.4 | 跳过 reparse point（junction/symlink）、系统/隐藏文件可配 | 不跟随符号链接、不死循环 |
| FR-2.5 | 路径规范化：统一 `/` 分隔、大小写不敏感键、非法字符（`:*?"<>|`）预检 | 非法文件名记为 `Skip invalid-name`，不导致整轮失败 |
| FR-2.6 | 改名/移动识别：同批次「1 删 + 1 增」且 size+hash 相同 → 视为 Move | 本地重命名 10 个文件，远端执行 rename 而非重新传输（`bytes=0`） |
| FR-2.7 | **USN 增量监控（本地 NTFS 可选加速）**：以管理员身份读取本地卷 USN Journal，持久化 `(journalId, nextUsn)` 检查点，每 1–2s 读增量，仅对变化路径 stat；期刊失效（`FirstUsn` 越界 / journalId 变更 / 无权限 / 非 NTFS）→ 自动降级 FSW 并全量重建 | 有权限时 `monitor-mode=usn`，改名 1000 文件产 1000 个 Move 且 `bytes=0`；删除期刊后自动降级且不漏同步；无权限时 `monitor-mode=fsw` 功能完整 |
| FR-2.8 | **扫描内存 O(1)**：扫描结果流式写入 SQLite TEMP staging 表，用 SQL 做集合归并产出“事实流”，C# 纯函数 `Decide()` 逐行决策（不在内存中构建清单） | 10 万文件索引峰值 < 50MB 且与文件数无关；v1 内存实现与 v2 SQL 归并实现的计划逐条相同（等价性测试） |
| FR-2.9 | **目录级合并遍历（merge-walk，默认模式）**：A/B 同名目录并行下钻，逐个目录 diff + 执行 + 落库，内存只与“最宽目录”相关；跨目录改名用**有上限**的候选索引（默认 2 万条） | 100 万文件树、单目录 ≤ 2000 条时，扫描期托管堆 ≤ 1MB；`diff.mode=merge` 为默认，`sql`/`memory` 可回退 |
| FR-2.10 | **能力档案探测**：启动时探测两侧 `Kind/Fs/HasUsn/AtomicRename/mtime容差/FileId`，写入 `status` 与日志；判定逻辑不随档位改变 | NAS 侧自动识别为 `WatcherReliable=false`、mtime 容差 2s、兜底 5min；本地同卷为 `AtomicRename=true` |

### FR-3 双向同步判定（P0）

| ID | 需求 | 验收标准 |
|---|---|---|
| FR-3.1 | 严格按三方比较（base/local/remote）产生动作：Push/Pull/DeleteLocal/DeleteRemote/Move/**MoveDir**/UpdateBase/Conflict/None | 12 分支表驱动单测全绿；目录改名走 `MoveDir`（同卷 rename，不整树重传）；手工构造每种情形计划正确 |
| FR-3.2 | 方向可配：`TwoWay` / `PushOnly` / `PullOnly` | PushOnly 模式下远端改动被判为 `Skip (direction)` 且有日志 |
| FR-3.3 | 先产出完整计划再执行；`--dry-run` 不产生任何写操作 | dry-run 前后文件 hash/时间戳不变 |
| FR-3.4 | 计划-执行分离，执行顺序：建目录(浅→深) → Move → Push/Pull → 删除(深→浅) → 落库 | 嵌套目录删除/创建不报错 |
| FR-3.5 | 单文件失败不中断整轮，进重试队列（5s/30s/2min…最多 5 次） | 注入一个只读目标文件 → 其余文件照常完成，失败项日志 `Error` |

### FR-4 实时监听与"编辑完成"判定（P0）

| ID | 需求 | 验收标准 |
|---|---|---|
| FR-4.1 | 两侧各一个 `FileSystemWatcher`，递归监听，`InternalBufferSize=64KB` | 子目录变更可捕获 |
| FR-4.2 | 事件按路径合并（防抖），静默 `debounceQuietMs`（默认 3000）后才进入探测 | 批量解压 1 万文件时内存不涨、队列 ≤ `queueCapacity` |
| FR-4.3 | 探测：`~$` 锁文件存在 → 挂起；`FileShare.None` 打开失败 → 退避重试（1s→30s，总超时 30min）；成功后复核 size/mtime 稳定 1s | S3 场景通过；日志含 `locked-by-office` / `in-use` 与等待时长 |
| FR-4.4 | 缓冲溢出 / Watcher Error → 立即触发一次增量兜底扫描并记 `Warn` | 人为制造溢出（大量快速写）后自动补齐 |
| FR-4.5 | 定时兜底扫描：默认 5 分钟增量、10 分钟快哈希校验（可配 0 关闭） | 断网期间产生的改动，恢复后在 5 分钟内被补齐 |
| FR-4.6 | self-write 防回环：写入前登记 `(path, ts, hash)`，15s 内同路径事件丢弃 | 同步完成后 60s 内日志无新增传输 |
| FR-4.7 | 监控模式自检与上报：启动探测（提权 / NTFS / 期刊 / 检查点）决定 `usn` 或 `fsw`，写日志并在 `status` 显示；运行期失效时降级并告警 | `status` 输出 `monitor-mode=usn|fsw` 与最近一次 USN；降级有 Warn 日志 + 原因 |

### FR-5 冲突处理（P0）

| ID | 需求 | 验收标准 |
|---|---|---|
| FR-5.1 | 冲突默认 `KeepBoth`：保留远端原样，本地副本命名 `name.conflict-yyyyMMdd-HHmmss-Local.ext`（反向同理） | 冲突后两端总文件数 = 原文件数 + 1，两份内容均可打开 |
| FR-5.2 | 可切换策略：`PreferNewer` / `PreferLocal` / `PreferRemote` | 各策略有单测；`PreferNewer` 以 mtime 大者胜，平局走 KeepBoth |
| FR-5.3 | 冲突必须在日志与 `conflicts` 子命令中可查 | `conflicts` 列出路径、两侧 mtime/size、产生时间 |
| FR-5.4 | 冲突不阻塞其他文件 | 单轮多个冲突全部处理完，退出码 2 |
| FR-5.5 | **冲突判定必须三方确认**：`localChanged && remoteChanged && 内容不同`，且“内容不同”必须由**全量 SHA-256 不相同**坐实（元数据差异不算），删除 vs 修改也算冲突 | 构造“只改 mtime 不改内容”的两端操作 → 判定为 `UpdateBase`，0 个冲突；构造真冲突 → 恰好 1 个，两份内容均可打开 |
| FR-5.6 | 冲突产物：胜者保留原名，败者在该侧改名为 `name.conflict-yyyyMMdd-HHmmss-<side>.ext`；同时写入 `conflicts` 台账（两侧 mtime/size/hash） | 两端目录列表都能看到两份文件；台账可用 `conflicts` 列出并标记已解决 |

### FR-6 删除保护（P0）

| ID | 需求 | 验收标准 |
|---|---|---|
| FR-6.1 | 每轮执行前探针：远端 root 可枚举（`EnumerateFileSystemEntries(...).Take(1)`）；失败 → 整轮跳过 | 断开 Z: → 0 个本地文件被删，退出码 4 |
| FR-6.2 | 删除阈值：单轮 `Delete*` > `maxDeletesPerRun`(100) 或占比 > `maxDeleteRatio`(25%) → 中止，退出码 3，需 `--force-deletes` | 手工删 30% 文件 → 中止且无删除动作 |
| FR-6.3 | `deletePolicy`：`Recycle`（默认，移入 `data\trash\<pair>\<ts>\`） / `Propagate`（真删） / `Ignore`（不传播删除）；**目录删除必须展开为逐条决策**（子树内有相对基线变过的文件 → 不删，逐个 Conflict/保留） | Recycle 模式下可从 trash 找回；目录删除不会整树盲删 |
| FR-6.4 | 两端都删 → 仅清理记录，不做任何文件操作 | 无垃圾日志、无 trash 产生 |

### FR-7 日志与审计（P0）

| ID | 需求 | 验收标准 |
|---|---|---|
| FR-7.1 | 结构化日志双写：**journal `LogEntry`**（可用 `logs` 子命令查询）+ JSONL 滚动文件（人读/可 grep） | 每条记录含 `run_id/op/direction/bytes/duration_ms/hash/message` |
| FR-7.2 | 一次运行一个 `run_id`，可整轮回放 | `logs --run <id>` 输出该轮全部条目 |
| FR-7.3 | 检索：`--since/--until/--path(glob)/--op/--level/--limit` | 组合过滤正确，`--path "docs/*"` 生效 |
| FR-7.4 | 保留策略：库中 90 天 / 200MB，文件日志 20MB × 14；超限按时间淘汰 | 达到上限后自动清理，不涨破 |
| FR-7.5 | 脱敏：日志只存相对路径；凭据永不落日志 | 全文搜索日志无密码、无完整 UNC（可配） |
| FR-7.6 | `status` 输出：待同步数、上轮统计（成功/失败/跳过/冲突）、远端连通性、失败重试队列 | 输出可被 `--json` 机器消费 |

### FR-8 运维与常驻（P1）

| ID | 需求 | 验收标准 |
|---|---|---|
| FR-8.1 | 支持前台 (`watch --foreground`) 与 Windows 服务/计划任务两种形态 | 服务方式下（指定有 NAS 权限的账户 + UNC 路径）能正常同步 |
| FR-8.2 | 崩溃/断电恢复：启动清理残留 `.filesync-tmp`，按 `inflight` 表重传未完成项 | kill 进程后重启：无残留 tmp，1GB 文件正确重传 |
| FR-8.3 | NAS 重连：凭据丢失时用 `WNetAddConnection2` 重建（可选功能，凭据存 DPAPI/凭据管理器） | 重启后无需手工 `net use` |
| FR-8.4 | 单实例保护（互斥量），避免重复运行打架 | 第二次启动直接退出，提示已有实例（退出码 6） |
| FR-8.5 | 优雅关闭：收到 Ctrl+C / Stop 时完成当前文件（或留在 inflight）后退出 | 关闭耗时 < 5s（无大文件时），状态可续 |

### FR-9 CLI 契约（P0）

```
FileSync.Host.exe <command> [options]

  init      --pair <name> [--ui] [--initial=keep-both|prefer-a|prefer-b|new-only|skip]
            [--confirm] [--dry-run] [--report <path>] [--detail differing|all]
  sync      --once [--pair <name>] [--dry-run] [--force-deletes] [--verbose] [--path <glob>]
  watch     [--foreground] [--pair <name>]
  status    [--json]
  logs      [--since <ts|7d>] [--until <ts>] [--path <glob>] [--op a,b] [--level x]
            [--run <id>] [--limit 200] [--json]
  conflicts [--json] [--resolve <path> --keep local|remote|both]
  rules     test <path...>          # 验证排除规则命中情况
  config    check | show
  version
```

`--dry-run` 输出样式（行首固定动词，便于 grep/awk）：

```
PLAN  run=20250101-100004  local=D:\Work\Docs  remote=\\NAS\share\Docs
  PUSH       docs\a.docx        1.2MB   local 2025-01-01 09:58:02
  PULL       docs\b.xlsx        3.4MB   remote 2025-01-01 10:00:01
  MOVE       docs\c.txt -> docs\c2.txt
  CONFLICT   docs\d.pdf         both changed since 2025-01-01 09:00:00
  DELETE_LOCAL docs\old\e.tmp   (recycled -> data\trash\local-cd\20250101-100004)
  SKIP       docs\f.docx        locked-by-office
  TOTAL 6 actions  push=1 pull=1 move=1 conflict=1 delete=1 skip=1  bytes=4.6MB
```

### FR-10 首轮差异确认 UI（自绘，可选，P1）

| ID | 需求 | 验收标准 |
|---|---|---|
| FR-10.1 | `init --ui` 启动独立 `FileSync.Ui.exe`（Win32+GDI 自绘，**无任何 UI 框架**）；通过 named pipe 取 `stats` / `page(offset,limit)`，只持当前页 ≤50 行 | UI 进程工作集 4–7MB；10 万条差异下滚动内存无增长 |
| FR-10.2 | 界面展示 6 类统计 + 差异明细（虚拟列表）+ 选中项两侧 mtime/size/hash；键盘 `1–5` 选择、`Enter` 确认、`Esc` 退出 | 键盘可完成全流程，鼠标仅辅助 |
| FR-10.3 | 破坏性选择（`PreferA/B`、大删）弹**自绘**确认条，需再按 `Y`；提交前引擎**校验清单指纹**，已变则拒绝并要求重新清点 | 指纹变更时提交被拒（退出码 8），无文件被改 |
| FR-10.4 | UI 不可用/不带 `--ui` 时，`init` 的 CLI 菜单必须与 UI **完全等价**（同一决策枚举、同一审计记录） | 两种入口产生相同 `InitDecision` 与相同计划 |
| FR-10.5 | UI 崩溃/关闭不影响引擎；决策只在引擎侧落盘 | kill UI 进程后 `init` 状态一致 |

## 6. 非功能需求（NFR）

| ID | 类别 | 指标 |
|---|---|---|
| NFR-1 | 内存 | **稳态工作集 < 40MB；复制 1GB 文件峰值 < 50MB；10 万文件索引峰值 < 50MB**；占用必须与文件数量、单文件大小无关（O(1)）；复制期间 Gen2 GC = 0 次；`System.GC.HeapHardLimit` 作兜底 |
| NFR-2 | 内存安全 | 任何代码路径不得整文件读入内存；缓冲区走 `ArrayPool`；事件队列为定长 ring（8192，可配）且溢出有降级策略 |
| NFR-3 | 吞吐 | 千兆局域网：本地盘 → SMB 复制吞吐 ≥ 60MB/s（受 NAS 与网络限制）；不引入额外的双倍读写 |
| NFR-4 | 延迟 | 普通文件事件→同步完成 P95 < 8s；被占用文件在释放后 `probeTimeoutMs` 内完成 |
| NFR-5 | CPU | 空闲（仅监听+5min 扫描）平均 CPU < 1%；同步时不占满所有核心（并发 ≤ 2，可配） |
| NFR-6 | 可靠性 | 任意时刻 kill -9 → 不产生损坏的正式文件（只可能留下 `.filesync-tmp`，启动清理）；状态库事务保证"文件已落盘 ⟺ 基线已更新" |
| NFR-7 | 数据安全 | 默认不真删（进 trash）；删除阈值保护；dry-run 零写入；日志不含凭据 |
| NFR-8 | 可维护 | Core 层不依赖框架，可对内存假文件系统做单测；关键判定分支覆盖率 ≥ 90% |
| NFR-9 | 兼容 | Windows 10 1809+ / Windows 11；NTFS 本地；SMB2/3；长路径 > 260 字符可用（`\\?\`） |
| NFR-10 | 部署 | 单文件自包含 exe（引擎约 12–18MB，**不含 SQLite native**）；可选 UI 为独立小 exe（约 200KB）；`data\` 可指定路径且必须在所有 root 之外 |

## 7. 数据模型

> 已定：默认 `state.provider=journal`（纯托管、无 SQLite）。SQLite 仅为可选 provider，不得作为唯一实现。

### 7.1 journal 记录（`data\state.journal` + `state.snapshot`）

追加式记录：`[u32 len][u32 crc32][payload]`；重放遇 CRC 不符 / 长度越界即截断到上一条（等价于“最后一次成功提交”）。journal 超快照 2 倍或超 4MB 时压实（写新 snapshot + 原子 rename + 清空 journal）。

| 记录类型 | 作用 | 关键字段 |
|---|---|---|
| `Baseline` | 每相对路径 = 上次成功同步的基线 | `rel, kind, size, mtimeA, mtimeB, hash8, hash32?, op, ts, fileId?` |
| `Tombstone` | 一端删除、等待对端确认 | `rel, side, ts` |
| `Pending` | 首轮 `NewOnly`/`Skip` 挂起的差异项（**防止以后自动变冲突**） | `rel, reason, ts` |
| `LogEntry` | 事实日志 | `ts, level, op, rel, bytes, durationMs, hash, message, runId` |
| `Inflight` | 断电恢复（未完成的复制） | `rel, direction, tempPath, startedAt` |
| `ConflictNote` | 冲突台账 | `rel, conflictPath, side, ts` |
| `UsnCheckpoint` | USN 续读点 | `volId, journalId, nextUsn, updatedAt` |
| `DirtyTodo` | 崩溃重放用的脏目录待办 | `side, dirRel, reason, ts` |
| `InitDecision` | 首轮决策审计 | `pair, choice, fingerprint, ts, runId` |

写入纪律：
1. **`文件落盘成功 ⟺ 基线已更新`**：`Baseline` 与 `LogEntry` 写在同一批追加里，先 `fsync` 再推进内存索引。
2. 只存元数据（10 万条 ≈ 7MB 内存 / 每条 69B），不存内容。
3. **USN 顺序续订**：先把 `DirtyTodo` fsync，再写 `UsnCheckpoint`（反之会永久丢事件）。
4. manifest / sqlite 两种 provider 必须产出与 journal **逐条相同**的计划（等价性测试）。

### 7.2 目录布局与回收区

```
data\                      # 必须在所有 root 之外（启动校验），否则会自己同步自己
├─ state.snapshot          # 定长基线快照 + 路径 arena
├─ state.journal
├─ logs\fs-YYYYMMDD.jsonl
├─ trash\<pair>\<yyyyMMdd-HHmmss>\...   # 保留 30 天
└─ reports\firstrun.idx / firstrun.paths
```

## 8. 交互与输出契约

### 8.1 退出码

| 码 | 含义 |
|---|---|
| 0 | 成功（可能含 Skip，无失败） |
| 2 | 存在失败项或冲突（已处理，需人工关注） |
| 3 | 删除保护触发，未执行删除 |
| 4 | 远端不可用 / 探针失败，本轮跳过 |
| 5 | 配置错误 |
| 6 | 已有实例在运行 |
| 7 | 状态存储不可用（journal 不可写 / 磁盘满） |
| 8 | 首轮需人工决策：无 TTY 且未指定 `--initial=` |
| 9 | 需要 `--confirm` 的破坏性操作被拒绝执行（首轮 PreferX / 大删 / `--force-deletes` 缺失） |

### 8.2 关键配置项（默认值即产品契约）

| 键 | 默认 | 说明 |
|---|---|---|
| `conflictPolicy` | `KeepBoth` | 冲突策略 |
| `deletePolicy` | `Recycle` | 删除策略 |
| `direction` | `TwoWay` | 同步方向 |
| `realtime.debounceQuietMs` | 3000 | 静默窗口 |
| `realtime.probeTimeoutMs` | 120000 | 单文件探测超时 |
| `realtime.maxWaitMinutes` | 30 | 被占用文件最长等待 |
| `realtime.fullRescanMinutes` | 10 | 兜底全量校验间隔（0=关） |
| `performance.maxParallelTransfers` | 2 | 并发传输 |
| `performance.bufferBytes` | 131072 | 复制缓冲（128KB） |
| `performance.queueCapacity` | 8192 | 事件 ring 容量（有界，溢出置 NeedFullRescan） |
| `performance.hashMode` | `Quick` | `Metadata`/`Quick`/`Full` |
| `safety.maxDeleteRatio` / `maxDeletesPerRun` | 0.25 / 100 | 删除阈值 |
| `logging.dbRetentionDays` / `dbMaxMb` | 90 / 200 | 日志保留 |
| `monitoring.mode` | `auto` | `auto`（优先 USN，失败降级 FSW）/`usn`/`fsw`；**已决定允许提权，默认走 USN** |
| `state.provider` | `journal` | **已定：journal**（纯托管、无 SQLite）/`manifest`/`sqlite` |
| `monitoring.usnPollMs` | 1000 | USN 轮询间隔（决定丢事件风险 vs 开销） |
| `monitoring.fallbackFullScanMinutes` | 30 | USN 生效时的全量校验频率 |
| `diff.mode` | `merge` | `merge`（目录级合并遍历，O(最宽目录)）/ `sql`（staging+SQL 归并，O(1)）/ `memory`（v1 兼容，回退用） |

## 9. 错误与告警

| 事件 | 级别 | 处理 |
|---|---|---|
| `remote-unavailable` | Error | 跳过本轮，退避重连，退出码 4 |
| `delete-guard-triggered` | Error | 中止删除，退出码 3，提示 `--force-deletes` |
| `in-use` / `locked-by-office` 超时 | Warn | 本轮跳过，下轮兜底重试 |
| 单文件复制失败 | Error | 重试队列（最多 5 次），退出码 2 |
| `watcher-overflow` | Warn | 立即增量兜底扫描 |
| `state-db-locked` | Error | 单实例互斥 + 15s 重试，仍失败退出码 7 |
| `invalid-name` / `path-too-long` | Warn | 跳过该文件，不中断 |
| 磁盘空间不足（目标端） | Error | 中止本轮，保留 inflight，提示清理 |

## 10. 依赖与硬约束（来自环境实测）

1. **本机只有 .NET SDK 8.0.406** → 目标框架必须 `net8.0`，否则无法构建。
2. **`Z:` 是每用户会话映射**（实测当前会话 `net use` 为空，无 Z:）→ 服务/计划任务必须指定有 NAS 权限的账户，且**推荐直接配置 UNC 路径**；盘符在启动时用 `WNetGetConnection` 解析。
3. **SMB 上的 `FileSystemWatcher` 会丢事件**（服务端缓冲、断线、休眠）→ 必须有定时兜底扫描，实时性不能作为唯一保证。
4. **跨 SMB 的 mtime 精度可能被取整**（1–2s）→ 时间比较留 ±2s 容差，最终以哈希定论。
5. **NAS 侧可能同时有人手工改文件** → 只能"检测冲突 + 保留双份"，不能承诺强一致。
6. 中国区网络下 NuGet 可达（实测 `api.nuget.org` 正常），依赖仅 4 个包：`Microsoft.Data.Sqlite`、`Serilog`、`Serilog.Sinks.File`、`Serilog.Sinks.Console`。

## 11. 验收标准

### 11.1 总验收（全部通过才可交付）

| # | 验收项 | 通过判据 |
|---|---|---|
| A1 | 双向正确性 | 12 分支判定表全部符合；两端修改后各自收敛 |
| A2 | 编辑中文件 | S3 场景：编辑期间 0 次同步，关闭后 1 次同步且内容为最终版 |
| A3 | 冲突 | 5 种策略行为符合 §FR-5；两份内容均可打开 |
| A4 | 删除安全 | 断连 30 分钟 0 删除；超阈值中止；trash 可找回 |
| A5 | 内存 | 引擎稳态 < 40MB；1GB 文件复制峰值 < 50MB；10 万文件索引 < 50MB；UI 进程 4–7MB；无 OOM |
| A6 | 日志 | 每条动作可查；双写一致；保留策略生效 |
| A7 | 恢复 | kill -9 后重启无残留 tmp、正确续传；重复启动被拒（退出码 6） |
| A8 | 回环 | 同步完成后 60s 内无重复传输 |
| A9 | 部署 | 服务/计划任务形态下无人值守运行 24h 无异常、无内存增长趋势 |

### 11.2 测试矩阵（实现阶段必须落地）

| 类别 | 用例 |
|---|---|
| 单元 | DiffEngine 12 分支 × (File/Dir) × (有无 hash) |
| 单元 | IgnoreRules glob 边界（`**/`、`*`、大小写、路径分隔符） |
| 集成 | 1KB / 10MB / 2GB 文件传输正确性 + 内存断言 |
| 集成 | 本地与远端模拟（临时目录）——CI 无需真实 NAS |
| 集成 | 改名 / 目录移动 / 大量小文件（10k） |
| 集成 | 冲突 5 策略、删除 3 策略 |
| 集成 | 断网、只读目标、磁盘满、非法文件名、超长路径 |
| 手动 | 真实 Z: 上跑 dry-run → 校验计划 → 实跑 |
| 手动 | Word/WPS 编辑保存（S3）、Excel、压缩大包（事件洪泛） |
| 长稳 | 24h 常驻 + 每 10 分钟改动，观察内存曲线与日志量 |

## 12. 里程碑

| 阶段 | 内容 | 工期 | 出口条件 |
|---|---|---|---|
| M0 | 骨架、配置/校验（含 data 目录与跨 pair 重叠）、journal 追加+CRC+压实、CLI 框架 | 1d | `status` 可用；kill -9 后重放正确 |
| M1 | merge-walk + 三级指纹 + 单缓冲复制 + 原子落盘 + 重试 | 1.5d | 1GB 复制峰值 < 50MB；无残留 tmp；改名零传输 |
| M2 | 双向判定 + 基线/墓碑 + 冲突 + trash + inflight | 1.5d | 12 分支单测全绿；A1/A3/A5 通过 |
| M3 | 首轮识别（6 类分桶）+ 五选一 + 指纹校验 + 报告 | 1d | FR-1.7~1.9 通过；`--dry-run` 零写入 |
| M4 | 监控：FSW + USN + DirtySet + 稳定化闸门 + 对账 | 1.5d | A2/A4/A8 通过；USN 失效自动降级 |
| M5 | `init --ui` 自绘窗口（统计/虚拟列表/确认条/分页 IPC） | 1.5d（可选） | UI 4–7MB；与 CLI 决策等价 |
| M6 | 服务化（LocalSystem）+ 24h 长稳 + 压测 | 0.5d | A7/A9 通过 |
| **合计** | | **≈8 天**（含 UI） / ≈6.5 天（不含 UI） | |

## 13. 风险与对策

| 风险 | 概率 | 影响 | 对策 |
|---|---|---|---|
| SMB 事件丢失导致"不同步" | 高 | 中 | 5/10 分钟兜底扫描 + 溢出立即扫描 |
| 服务看不到映射盘 | 高 | 高 | UNC 一等公民 + 指定账户运行（§10-2） |
| NAS 离线被当成全删 | 中 | **致命** | 探针 + 删除阈值 + 默认 Recycle |
| 编辑中文件读到半成品 | 中 | 高 | 独占探测 + 稳定复核 + `~$` 感知 + 原子 rename |
| 两端同时改丢数据 | 中 | 高 | KeepBoth 默认 + 冲突台账 |
| 海量小文件扫描慢 | 中 | 中 | 流式枚举 + 批量事务 + 元数据优先 |
| NAS 为 Linux 共享：大小写/非法字符 | 中 | 中 | 大小写不敏感键 + 非法名预检跳过 |
| 内存膨胀 | 低 | 高 | 全流式 + ArrayPool + 有界队列 + 压测门槛 |
| 用户误以为"实时"等于"强一致" | 中 | 中 | 文档与 `status` 明确展示"最后扫描时间/待同步数" |

## 14. 待确认问题（评审需拍板）

| # | 问题 | 建议默认 | 影响 |
|---|---|---|---|
| Q1 | `remoteRoot` 用 UNC（`\\NAS\share\x`）还是盘符 `Z:`？ | **UNC** | 决定服务形态可行性 |
| Q2 | 冲突默认策略？ | `KeepBoth` | 决定冲突副本命名与用户心智 |
| Q3 | 删除策略：真删还是进 trash（保留 30 天）？ | `Recycle` | 影响磁盘占用与安全 |
| Q4 | 是否需要托盘/最小 GUI（v1 或 v2）？ | v2 | 影响工期约 +1.5d |
| Q5 | 日志保留 90 天 / 200MB 是否合适？NAS 侧是否也需留存？ | 90 天 / 仅本地 | 影响磁盘与合规 |
| Q6 | **是否允许同步器以管理员 / LocalSystem 身份运行？**（USN 读取的前置条件；实测非管理员 `GENERIC_READ` 打开卷 → 拒绝访问） | **✅ 已定：允许** → USN 启用 | 决定本地侧能否“零丢事件 + O(变化数)扫描”，是 v2 优化落地的开关 |
| Q7 | 状态后端默认用哪种？`journal`（纯托管、无依赖）/ `manifest`（本地双盘 O(1)）/ `sqlite`（可 SQL 查日志） | **✅ 已定：`journal`** | 影响 exe 体积、启动成本、日志检索方式 |

## 15. 参考项目（调研结论，供评审判断"为什么不直接用"）

| 项目 | 借用点 | 不用的原因 |
|---|---|---|
| syncthing/syncthing（90.7k★） | 增量扫描、rescan、冲突副本命名 | 需 NAS 端 agent，不走 SMB 映射 |
| deajan/osync（998★） | 计划-执行两遍收敛、删除保护 | Shell + rsync，无 Windows 状态库 |
| POW-Software/ByteSync（177★, C#/MIT） | 分层结构、Inventory 清单、临时文件策略 | 面向跨站点 Delta 协议，过重 |
| LINBIT/csync2（163★） | "每文件一条 last-sync 状态"的核心模型 | C + librsync，无 Windows 实现 |
| no-src/gofs（526★） | 事件驱动 + 防抖 + 定时兜底 | Go，且同样缺状态库语义 |
| FreeFileSync / robocopy /MIR | 快速验证基线行为 | 无三方基线，双向会静默覆盖 |
