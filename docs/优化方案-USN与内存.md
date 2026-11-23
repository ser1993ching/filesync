# USN 与内存优化（v3 规范入口）

> 本文原有内容中的 O(1) 全量索引、quick hash 证明不变、默认提权/服务运行等说法不能直接作为实现要求，现统一由 v3 规范替代。

当前原则：
- USN 只服务于本地 NTFS 增量发现；必须绑定 VolumeId、JournalId 和 NextUsn checkpoint。
- JournalId 改变、USN 记录范围失效、读取失败或权限不足时，立即失去增量可信度并触发对账。
- FSW/USN 事件队列必须有界；溢出标记 NeedReconcile，不得静默丢弃。
- NAS 不使用 USN；NAS 内容变化依赖 SMB 事件提示、周期对账和内容审计。
- Metadata、Quick fingerprint 只能筛选；完整 SHA-256 才用于内容等价判断。
- GC HeapHardLimit 仅限制托管堆，不能代表进程 Working Set、Private Bytes、Native Memory 或内核 Nonpaged Pool。
- 低内存必须通过固定测试环境和可复现报告验收，不能只看单次进程工作集。

详细规范见：
- [低内存存储与扫描架构](v3-低内存存储与扫描架构.md)
- [同步引擎与三方状态机](v3-同步引擎与三方状态机.md)
- [测试策略与可靠性验收](v3-测试策略与可靠性验收.md)
