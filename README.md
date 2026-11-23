# FileSync

Windows 上的**双目录双向实时同步工具**（本地盘 ⇄ 本地盘 / 本地盘 ⇄ NAS 映射盘）。

- 运行时：.NET 8（单文件自包含 exe，无 SQLite、无 UI 框架）
- 形态：CLI + Windows 服务（LocalSystem）+ 可选自绘 UI 进程
- 目标：引擎稳态内存 < 40MB（与文件数无关）、改完即同步、冲突不丢数据、全程可审计

> 当前状态：**设计定稿，尚未开始编码**。本仓库目前只包含设计文档。

---

## 文档索引（建议按此顺序阅读）

| 文档 | 作用 |
|---|---|
| [`docs/PRD.md`](docs/PRD.md) | **需求唯一来源**：目标/非目标、用户场景、FR-1~FR-10、NFR、数据模型、退出码、验收、里程碑、风险、待决问题 |
| [`docs/骨架图.md`](docs/骨架图.md) | **架构与流程图示**：进程/线程、分层、目录骨架、数据流、首轮流程、监控分层、状态机、判定矩阵、内存预算、部署 |
| [`docs/监控设计.md`](docs/监控设计.md) | 文件变化监控定稿：FSW + USN Journal + 对账、稳定化闸门、失效降级 |
| [`docs/优化方案-USN与内存.md`](docs/优化方案-USN与内存.md) | USN 实测（权限、期刊参数）、内存实测表（206B → 69B → O(1)）、merge-walk 预算 |
| [`docs/变体方案-无SQLite与本地双盘.md`](docs/变体方案-无SQLite与本地双盘.md) | 不用 SQLite 的三种状态后端、任意两路径对（含 `C:\A ⇄ D:\B`）、能力档案 |
| [`docs/UI方案-自绘低内存.md`](docs/UI方案-自绘低内存.md) | 首轮差异确认窗口：Win32+GDI 自绘、虚拟列表、mmap 分页、4–7MB |
| [`docs/技术方案.md`](docs/技术方案.md) | **v1 基线**（部分已被取代，文首有取代索引，实现以其余文档为准） |

## 已定关键决策

| 决策 | 结论 |
|---|---|
| USN Journal | 启用（允许以管理员/LocalSystem 运行） |
| 状态后端 | `journal`（追加日志 + 快照 + CRC，纯托管，无 SQLite） |
| 扫描模式 | `diff.mode=merge`（目录级合并遍历，内存 O(最宽目录)） |
| UI | 独立 `FileSync.Ui.exe`，Win32+GDI 自绘，无任何 UI 框架 |
| 删除 | 默认进 `data\trash\`，阈值熔断（25% / 100 个） |
| 冲突 | `KeepBoth` 默认，`.conflict-时间-侧` 副本 + 台账 |
| 首轮 | 只读清点 → 五选一（全保留/以A/以B/只同步新增/不同步）→ 确认后执行 |

## 计划中的目录结构

```
src/
├─ FileSync.Core/     扫描、指纹、判定、执行（纯逻辑，可单测）
├─ FileSync.State/    journal 状态后端（journal | manifest | sqlite 可选）
├─ FileSync.Host/     服务外壳：监控、合并、调度、IPC、首轮
└─ FileSync.Ui/       Win32 自绘窗口（可选）
tests/
├─ FileSync.Core.Tests/
└─ FileSync.Integration.Tests/
```
