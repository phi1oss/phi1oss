# SMMU Learning Hub

这是 Arm Memory Management、SMMU / MMU-720AE 学习的本地知识中枢。以后 ChatGPT 问答、培训材料与规范解析、正式知识、综合关系、面试材料和薄弱项都在这里维护。

当前学习阶段是 **Armv9 Memory Management**，主资料为 `99_SOURCE\Training\MMU.pdf` 的首个模块和配套培训视频截图。SMMU/MMU-720AE TRM 是后续阶段与实现背景。

## 三个系统的职责

| 系统 | 职责 | 不承担 |
|---|---|---|
| 本地 Learning Hub | 保存完整知识正文、来源、综合关系和复盘记录，是知识真相源 | 不负责提醒和任务排期 |
| Linear | 管理长期主题、学习状态、成熟度、弱点和 Next Action | 不堆放全部长篇笔记 |
| 当前 ChatGPT 工作窗口 | 解析 TRM、整理问答、蒸馏知识、模拟面试、维护本地与 Linear 的对应关系 | 不擅自修改云端状态 |

## 本地与手机记录

- 默认且优先：写入本地 `D:\MY_LEARNING\SMMU_Learning_Hub`。
- 手机端无法访问本地目录时：该时间段临时写入 Google Drive。
- 两处不自动同步。用户隔一段时间手动更新并提醒后，再进行人工对齐。
- 每条新 Inbox 记录都写明记录日期和记录端，用于复盘和识别不同时间段。

## 目录怎么用

| 目录 | 放什么 | 典型输出 |
|---|---|---|
| `00_HOME` | 状态入口、原始问答索引和工作方法 | Current Status、记录索引、操作规范 |
| `01_INBOX` | 尚未蒸馏的原始输入 | ChatGPT 问答、TRM 片段、问题、想法 |
| `02_KNOWLEDGE` | 经过验证、可复用的正式知识 | 概念说明、机制解析、配置与调试知识 |
| `03_SYNTHESIS` | 跨主题连接与高层归纳 | 端到端流程、概念对比、架构图、关键洞察 |
| `04_INTERVIEW` | 面试表达与测试材料 | 精炼回答、题库、单文件 Assessment |
| `05_WEAKNESS` | 跨测试持续存在的知识缺口 | 长期错答模式、重学记录、待验证假设 |
| `98_ARCHIVE` | 不参与日常维护的历史记录 | 手机端导入日志、退役记录 |
| `99_SOURCE` | 原始资料 | TRM、Architecture Spec、Training、论文 |

## 最常用的指令

- “记录最近一次问答。” → 只读取另一个 `SMMU` 聊天最近一段完整问答，保存到 `01_INBOX\ChatGPT`，配置日期并更新索引。
- “把这个主题蒸馏成正式知识。” → 结合相关来源，更新 `02_KNOWLEDGE`。
- “整理 X 和 Y 的区别/完整流程。” → 输出到 `03_SYNTHESIS`。
- “生成面试版并测试我。” → 每次测试在 `04_INTERVIEW\Assessments` 生成一份完整记录；只有跨测试持续存在的问题才进入 `05_WEAKNESS`。
- “同步这个主题到 Linear。” → 更新已有 Issue 的结论、成熟度、弱点和 Next Action。
- “复盘本周学习。” → 更新 Current Status 中的培训、整理和测试进度。

## 入口

- [工作区操作方法](00_HOME/Workspace_Operating_Guide.md)
- [当前学习状态](00_HOME/Current_Status.md)
- [Linear 协作方法](00_HOME/Linear_Workflow.md)
- [ChatGPT 记录索引](00_HOME/ChatGPT_Record_Index.md)

现有 `00`–`62` 条 ChatGPT 记录已经进入 `01_INBOX\ChatGPT`。它们属于已捕获素材；经过归并、查证和测试后，才晋升为正式知识。测试原始作答不再重复进入 Inbox，而是保存在对应 Assessment 中。
