# FileSync v3.0：NAS 与本地文件系统适配层

> 状态：实现规范（Normative）
> 目标：让同步核心只依赖端点能力，不依赖 Windows API、SMB 细节或 UI。

## 1. 分层原则

- Core 只处理相对路径、观察结果、指纹、Base、冲突与 SyncPlan。
- Abstractions 定义端点和变化源接口。
- Platform.Windows 实现 NTFS、SMB/UNC、FSW、USN、凭据与 Windows 原生能力。
- Engine 编排扫描、计划、执行、重试和恢复。
- UI/Host 不得绕过 Engine 直接操作同步数据。

## 2. 推荐项目边界

~~~text
FileSync.Core
  SyncDecision / ThreeWayMerge / ConflictResolver / SyncPlan
FileSync.Abstractions
  IFileSystemEndpoint / IStateStore / IChangeSource / ILeaseProvider
FileSync.Storage
  Journal / Snapshot / SortedIndex / Recovery
FileSync.Platform.Windows
  LocalNtfsEndpoint / SmbEndpoint / UsnMonitor / FileWatcher
FileSync.Engine
  Scanner / Reconciler / Planner / Executor / RetryScheduler
FileSync.Host
  Lifecycle / TaskScheduler / CLI / IPC
FileSync.Ui
  Win32 + GDI
~~~

Core 不得引用 System.IO.FileSystemWatcher、P/Invoke、Windows Credential Manager 或 UI 程序集。可通过测试用 FakeEndpoint 注入各种端点行为。

## 3. IFileSystemEndpoint 契约

建议接口语义（名称可按代码风格调整，但能力不能丢失）：

~~~csharp
public interface IFileSystemEndpoint
{
    EndpointDescriptor Descriptor { get; }
    ValueTask<EndpointProbeResult> ProbeAsync(CancellationToken ct);
    IAsyncEnumerable<DirectoryEntry> EnumerateAsync(
        RelativePath directory, CancellationToken ct);
    ValueTask<FileMetadata?> GetMetadataAsync(
        RelativePath path, CancellationToken ct);
    ValueTask<Stream> OpenReadAsync(
        RelativePath path, CancellationToken ct);
    ValueTask<Stream> CreateNewTemporaryAsync(
        RelativePath path, CancellationToken ct);
    ValueTask FlushFileAsync(RelativePath path, CancellationToken ct);
    ValueTask<CommitResult> CommitAsync(
        CommitRequest request, CancellationToken ct);
    ValueTask MoveToRecycleAsync(
        RelativePath source, RelativePath recyclePath, CancellationToken ct);
    ValueTask DeleteAsync(RelativePath path, CancellationToken ct);
}
~~~

接口应返回结构化错误，不把所有异常都压成“文件不存在”。至少区分：
- NotFound（只有在端点状态健康且路径查询成功时成立）；
- AccessDenied；
- EndpointUnavailable / Timeout；
- SharingViolation / Locked；
- InvalidName / PathTooLong；
- OutOfSpace；
- UnsupportedCapability；
- TransientIO / PermanentIO。

## 4. EndpointDescriptor 能力档案

每个端点在创建任务和运行期间暴露：
- EndpointId、类型、规范化根路径、文件系统/服务器标识；
- 大小写敏感规则与名称比较规则；
- 时间戳精度与时区语义；
- 文件 ID 是否稳定、范围及重启持久性；
- 是否支持 USN、Change Notification；
- 是否支持原子创建、同目录重命名、原子替换；
- Flush 语义、文件锁行为、最大路径/名称长度；
- 文件名非法字符、保留名称、大小写碰撞行为；
- 可用空间、权限与只读状态。

能力探测结果要带版本与探测时间；能力未知按保守策略处理。不能把“某个 SMB NAS 支持”泛化为“所有 NAS 支持”。

## 5. 路径与名称语义

- 保存用户输入根路径、规范化根路径和端点原始路径三种信息。
- 内部相对路径使用统一分隔符，但必须保留原始文件名。
- 禁止全局一律使用 OrdinalIgnoreCase。路径比较必须使用端点声明的规则。
- 两端比较规则不兼容时，建立任务前给出风险提示；扫描发现大小写折叠冲突时停止该子树自动写入。
- Windows 不允许的文件名或 NAS 端无法表达的名称，标记为 UnsupportedName，不得擅自改名后覆盖另一文件。
- 尾随点/空格、设备保留名、Unicode 规范化、冒号/备用数据流、长路径都要有独立测试。
- 规范化和防路径穿越校验必须在所有读写操作前执行。

## 6. 本地 NTFS 端点

- 使用当前用户上下文访问用户目录和映射盘；不要默认 LocalSystem 能访问交互用户映射盘。
- UNC 是 NAS 连接的一等配置形式；映射盘仅作为用户会话中的路径别名。
- USN 只用于 NTFS 变化加速，不能替代完整性审计。
- USN checkpoint 必须绑定 VolumeId 与 JournalId；JournalId 改变、NextUsn 超出有效范围、读取失败或权限不足时，丢弃增量可信度并触发全量对账。
- 文件 ID 可用于识别重命名，但必须处理硬链接、复制、卷迁移和 ID 重用边界。

## 7. SMB/NAS 端点

- 不要求 NAS 安装 Agent；通过 SMB2/3 与 UNC 路径访问。
- SMB Change Notification/FSW 只是提示，不是可靠事件日志；断线、缓冲溢出、NAS 实现差异可能丢事件。
- NAS 必须有周期对账和内容审计；目录 mtime/条目数不构成跳过子树的充分证据。
- 远端不可访问、认证失败、权限不足、目录枚举中断，必须返回端点级错误或 Incomplete Scan，不得返回空目录冒充成功。
- SMB 凭据通过 Windows Credential Manager 或 DPAPI 保护；不得写入配置明文、日志、崩溃报告。
- 网络路径下原子替换、Flush、文件锁、时间精度等均需实机探测/验证；不支持的能力应导致安全降级或拒绝危险动作。

## 8. 多客户端任务租约

本地互斥量只阻止同一台机器上的重复进程，不能阻止另一台电脑操作同一任务。

第一版多客户端防护：
- 任务在 NAS 共享根或专用任务目录创建 Lease 文件；
- 创建必须使用不覆盖式原子操作；
- Lease 包含 TaskId、OwnerId、LeaseId、进程实例、心跳时间、过期时间和格式版本；
- 持有者定期续租，并在每次破坏性操作前检查 LeaseId；
- 过期租约接管前需二次确认旧持有者失活；时钟漂移、网络分区时采用保守策略；
- 释放租约时仅允许删除 LeaseId 匹配的文件；
- 如果 NAS 不支持可靠锁文件语义，则同一任务仅允许单客户端，UI 明确告知。

租约不是分布式事务，也不能阻止用户直接在 NAS 文件管理器修改文件；文件提交前仍需目标状态复核。

## 9. 错误映射与重试

重试仅针对可恢复错误，采用指数退避 + 随机抖动 + 最大次数/时间预算。AccessDenied、InvalidName、UnsupportedCapability 不应无限重试。EndpointUnavailable 期间暂停依赖远端缺失的判定和删除动作。

## 10. 适配层测试

FakeEndpoint 必须能模拟：
- 不排序枚举、大小写敏感/不敏感、时间精度差异；
- 不支持原子替换、Flush 不可靠、文件 ID 不稳定；
- 访问被拒、目录中途枚举失败、远端断线；
- 文件被其他进程并发修改、锁定；
- 磁盘满、短读短写、重命名失败；
- 租约竞争、租约过期、心跳丢失。

真实集成测试至少覆盖 Windows NTFS + 两种 SMB 服务端（如 Samba 与常见 NAS）并保留测试记录。
