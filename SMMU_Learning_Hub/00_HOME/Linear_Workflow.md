# Linear 与本地知识库协作方法

## 当前配置

- Workspace：`philo`
- Team：`Philo`（`PHI`）
- Project：[SMMU 学习资料库](https://linear.app/philo990611/project/smmu-%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93-19f31fbcad7a)
- Document：[SMMU / MMU-720AE 学习地图](https://linear.app/philo990611/document/smmu-mmu-720ae-%E5%AD%A6%E4%B9%A0%E5%9C%B0%E5%9B%BE-4de6fe48a98c)

## 职责边界

本地文件保存完整知识正文和证据，Linear 管理“接下来学什么、掌握到什么程度、哪里薄弱”。不要把同一篇长笔记在两个系统中重复维护。

一个 Issue 对应一个长期主题；一次 ChatGPT 问答通常只作为该主题的新证据或评论，不单独建 Issue。

Google Drive 只在手机无法访问本地 Hub 时临时承接该时间段的记录。它与本地目录不自动同步，也不会自动触发 Linear 更新；用户手动对齐并明确要求后再处理。

当前按 Training/MMU.pdf 顺序推进，已从 Armv9 基础进入 SMMUv3 编程、接口与 DTI；最新定位见 [Current Status](Current_Status.md)。Linear 中已有 Issue 保持原状态，不因本地正式文档或进度调整自动更新。

## Issue 状态语义

| Linear 状态 | 在学习系统中的含义 |
|---|---|
| `Backlog` | 已识别，但尚未进入近期计划 |
| `Todo` | 已选入下一学习周期 |
| `In Progress` | 正在阅读、整理或验证；同时最多 3 个 |
| `In Review` | 已形成正式知识，等待脱稿解释或追问测试 |
| `Done` | 已有本地正式文档，且通过自测/模拟面试 |
| `Canceled` | 不再需要或超出当前范围 |
| `Duplicate` | 与已有主题重复，合并到主 Issue |

## 成熟度标准

- `L0 — Seen`：见过术语，尚不能定义。
- `L1 — Define`：能给出基本定义和职责。
- `L2 — Explain`：能借助提示解释动机、边界和基本流程。
- `L3 — Connect`：能脱稿讲清端到端流程及相邻概念关系。
- `L4 — Apply`：能处理连续追问、配置推演、故障分析或实际集成问题。

成熟度写入 Issue 描述或最新进度评论，并附证据，不仅凭主观感觉提升。

## 每个 Issue 建议维护的内容

```markdown
## Goal
本主题最终要能够解释或解决什么。

## Scope
包含什么；明确不包含什么。

## Current Maturity
L0–L4，以及支撑判断的证据。

## Confirmed Conclusions
已经查证且稳定的结论摘要。

## Local Knowledge
正式知识文档和相关 Inbox 文件的本地路径。

## Weakness
当前说不清、容易混淆或曾经错答的点。

## Next Actions
下一步最多 1–3 个可执行动作。
```

## 什么时候同步 Linear

需要同步：

- 一个主题开始成为当前学习重点。
- 新结论改变了主题边界或完整流程。
- 正式知识文档已经创建或显著更新。
- 自测后成熟度发生变化。
- 新增了明确弱点或下一步行动。

不需要同步：

- 只是保存一条原始聊天记录。
- 只是修改 Markdown 排版或文件名。
- 结论仍是未经验证的猜测。

## 当前主题映射

| Issue | 长期主题 | 主要本地知识域 |
|---|---|---|
| `PHI-5` | SMMU、设备 DMA 与 StreamID 基础模型 | `02_KNOWLEDGE\Fundamentals` |
| `PHI-6` | TCU、TBU 架构与职责 | `Architecture` |
| `PHI-7` | DTI、LTI 与主要接口 | `Interfaces` |
| `PHI-8` | Stream Table、STE 与 Context Descriptor | `Translation` |
| `PHI-9` | Stage 1、Stage 2 与两阶段转换 | `Translation` / `Virtualization` |
| `PHI-10` | Command、Event 与 PRI Queue | `Software` |
| `PHI-11` | MMU-720AE 软件初始化 | `Software` / `Integration` |
| `PHI-12` | TLB、Walk Cache 与 Configuration Cache | `Translation` |
| `PHI-13` | PCIe ATS、PASID、PRI | `Interfaces` / `Virtualization` |
| `PHI-14` | GPC、GPT 与 RME / Realm | `Security` |
| `PHI-15` | FuSa、FMU 与错误报告 | `FuSa` |
| `PHI-16` | PMU、ELA 与调试/性能分析 | `Debug` |

新增主题前先搜索这些 Issue。只有主题边界确实独立，才建立新的长期 Issue。

## 同步动作约定

用户说“记录”时，只保存本地 Inbox；若用户明确说明正在手机端，则写入 Google Drive 的该时间段记录。用户说“同步 Linear”时，助理先展示或概述拟更新的 Issue、成熟度、Weakness 和 Next Actions，再执行相应更新；优先更新既有 Issue，避免重复。任何本地、Google Drive 与 Linear 之间的同步都不自动发生。
