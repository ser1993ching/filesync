# 无 SQLite 与双目录方案（当前决策索引）

> 当前已确认：默认且首版唯一实现为纯托管 Journal + Snapshot，不使用 SQLite。本文原有的 manifest/sqlite 多 Provider、SQL staging 等变体不再作为首版实现要求。

## 当前状态存储规则

- Journal 追加式记录，具备长度、版本、序号和校验和。
- Snapshot 通过临时文件写入、Flush、校验后切换 manifest；旧 Snapshot 在新版本确认前保留。
- 索引磁盘化、有序段/分页读取；内存只保留有界页缓存。
- Operation 与 Base 由单一状态提交者串行写入。
- 磁盘满、Journal 损坏、Snapshot 切换失败时停止新的破坏性操作。
- Schema 版本化，未知关键版本不得静默降级。
- 任何恢复都必须检查真实文件状态和 OperationId，不能无条件删除临时文件。

## 双端模型

A/B 是对等端点，默认双向同步；本地盘 ↔ NAS/SMB 的两个方向都是 P0。NAS 不需要安装 Agent。NAS 与本地端点能力差异由 IFileSystemEndpoint 描述。

详细规范：
- [低内存存储与扫描架构](v3-低内存存储与扫描架构.md)
- [文件操作事务与崩溃恢复](v3-文件操作事务与崩溃恢复.md)
- [NAS 与本地文件系统适配层](v3-NAS与本地文件系统适配层.md)
