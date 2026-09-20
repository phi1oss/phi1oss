# Current Status

> 更新日期：2026-09-20  
> 学习主线：按照 `MMU.pdf` 培训目录推进。  
> 说明：培训进度、内容整理和测试熟练度相互独立，不以文档数量代表掌握程度。

## 1. 培训学到哪里

| 培训模块 | PDF 页码 | 当前进度 |
|---|---:|---|
| Armv9 Memory Management | 2–31 | 已学习 |
| SMMUv3 Programming | 32–57 | 已学习 |
| MMU S3 Introduction and Topology | 58–68 | 已学习 |
| MMU S3 Interfaces | 69–84 | 已学习 |
| DTI Protocol Overview | 85–93 | 已学习至第 91 页 |
| MMU S3 Translation | 94–103 | 已完成正式整理；掌握待测试 |
| Caching 及后续模块 | 104–160 | 尚未开始 |

当前资料处理位置：**MMU S3 Translation，第 103 页；该模块已完成正式整理。**

DTI Protocol Overview 第 92–93 页仍待补齐；后续主模块从第 104 页进入 `Caching`。

## 2. 内容整理到哪里

当前已经形成：

- 正式知识文档：19 篇；
- 跨主题综合文档：2 篇；
- 正式整理边界：已完成 MMU S3 Translation 第 94–103 页；DTI Protocol Overview 第 92–93 页仍有缺口。

正式知识文档按照知识关系分为以下五个大主题。

### 2.1 Armv9 基础层

核心关系：

> VA 经过页表查询得到 PA；Descriptor 和 MAIR 决定内存属性与访问行为；地址翻译配置发生变化后，需要进行相应的 TLB 维护。

| 文档 | 培训对应 | 整理状态 | 掌握验证 |
|---|---|---|---|
| [ARMv9 MMU 基础模型](../02_KNOWLEDGE/Fundamentals/01_ARMv9_MMU基础模型.md) | 第 2–6 页 | 已有正式文档；两次测试已归档 | L2 稳定，完整口述待复测 |
| [AArch64 多级页表与地址翻译](../02_KNOWLEDGE/Translation/01_AArch64多级页表与地址翻译.md) | 第 2–31 页 | 已有正式文档 | 已测试：🟡 不稳定 |
| [AArch64 Translation Regimes](../02_KNOWLEDGE/Translation/02_AArch64_Translation_Regimes.md) | 第 2–31 页 | 已有正式文档 | 已测试：🟡 不稳定 |
| [Translation Descriptors 与 Memory Attributes](../02_KNOWLEDGE/Translation/03_Translation_Descriptors与Memory_Attributes.md) | 第 2–31 页 | 已有正式文档；已补充属性辨析 | 核心属性🟢，descriptor边界🟡 |
| [Stage 1 配置与 TLB 维护](../02_KNOWLEDGE/Software/01_Stage1配置与TLB维护.md) | 第 2–31 页 | 已有正式文档 | 已测试：存在🔴未掌握项 |
| [ARM MMU Fault 与定位](../02_KNOWLEDGE/Debug/01_ARM_MMU_Fault与定位.md) | 第 2–31 页 | 已有正式文档 | 基础Fault分类🟢 |

### 2.2 Programming the SMMU

核心关系：

> 软件建立内存中的 Stream/CD 表与消息队列；请求以 SID/SSID 选择 Translation Context，软件通过 CMDQ 维护 SMMU，并通过 EVTQ、GERROR 和 PRIQ 处理运行期反馈。

| 文档 | 培训对应 | 整理状态 | 掌握验证 |
|---|---|---|---|
| [Programming the SMMU：SMMUv3 软件编程主线](../02_KNOWLEDGE/Software/02_Programming_the_SMMU主线.md) | 第 42–57 页 | 2026-09-15 按培训模块重建；原 5 篇编程层文档已删除 | 待测试 |

### 2.3 拓扑与设备侧模型

核心关系：

> 数据访问路径和地址翻译控制路径可以采用不同组合；设备既可以把普通访问交给 TBU，也可以通过 ATS 获取并缓存地址翻译结果。

| 文档 | 培训对应 | 整理状态 | 掌握验证 |
|---|---|---|---|
| [MMU S3 Introduction and Topology 主线](../02_KNOWLEDGE/Architecture/01_MMU_S3组件与设备集成拓扑.md) | 第 58–68 页 | 2026-09-17 按培训模块重建 | 待测试 |
| [PCIe ATS 两种模式与 ATC 维护](../02_KNOWLEDGE/Interfaces/01_PCIe_ATS两种模式与ATC维护.md) | 第 50、63–65、89 页 | 2026-09-10 形成正式文档 | 待测试 |
| [SVA 绑定、PRI 与 DTI 页面恢复](../02_KNOWLEDGE/Interfaces/02_SVA绑定_PRI与DTI页面恢复.md) | 第 46、50–51、55–57、91 页 | 2026-09-10 形成正式文档 | 待测试 |

### 2.4 接口与集成

核心关系：

> 接口决定数据访问、页表访问和控制信息如何传递；Properties 描述接口能力；内存属性最终需要转换为相应的总线事务属性。

| 文档 | 培训对应 | 整理状态 | 掌握验证 |
|---|---|---|---|
| [MMU S3 Interfaces 主线](../02_KNOWLEDGE/Interfaces/03_TBU_TCU接口与三类Bypass.md) | 第 69–84 页 | 2026-09-17 按培训模块重建 | 待测试 |
| [ACE5-Lite Properties 与输出属性](../02_KNOWLEDGE/Interfaces/04_ACE5-Lite_Properties与输出属性.md) | 第 71–76 页 | 2026-09-10 形成正式文档 | 待测试 |
| [SMMU 低功耗、复位与连接顺序](../02_KNOWLEDGE/Integration/01_SMMU低功耗_复位与连接顺序.md) | 第 70、77、82、87–88 页 | 2026-09-10 形成正式文档 | 待测试 |
| [PMCG 与 PMU Snapshot](../02_KNOWLEDGE/Debug/03_PMCG与PMU_Snapshot.md) | 第 80 页 | 2026-09-10 形成正式文档 | 待测试 |
| [AxCACHE、AxDOMAIN 与属性转换边界](../02_KNOWLEDGE/Interfaces/08_AxCACHE_AxDOMAIN与属性转换边界.md) | 第 16–18、71–76 页 | 2026-09-10 形成正式文档 | 待测试 |

### 2.5 DTI 协议层

核心关系：

> DTI 在 TBU、TCU 和 ATS requester 之间传递翻译、失效、同步、寄存器访问和页面请求信息；Token 管理容量，BAS 提供物理承载和路由。

| 文档 | 培训对应 | 整理状态 | 掌握验证 |
|---|---|---|---|
| [DTI 协议、连接与 Token 流控](../02_KNOWLEDGE/Interfaces/05_DTI协议_连接与Token流控.md) | 第 81–82、85–88 页 | 2026-09-10 形成正式文档 | 待测试 |
| [BAS 互连与 TID/TDEST 映射](../02_KNOWLEDGE/Interfaces/06_BAS互连与TID_TDEST映射.md) | 第 82–83、86 页 | 2026-09-10 形成正式文档 | 待测试 |
| [DTI 翻译响应、失效同步与寄存器访问](../02_KNOWLEDGE/Interfaces/07_DTI翻译响应_失效同步与寄存器访问.md) | 第 88–91 页 | 2026-09-10 形成正式文档 | 待测试 |

### 2.6 MMU S3 Translation 与保护

核心关系：

> SEC_SID 选择安全资源 bank，SID/SSID 通过 STE/CD 选择 Stage 1/2 Translation Context；地址翻译得到 PA 后，GPT/GPI 再检查对应 PAS 是否允许访问目标 granule。

| 文档 | 培训对应 | 整理状态 | 掌握验证 |
|---|---|---|---|
| [MMU S3 Translation 主线](../02_KNOWLEDGE/Translation/04_MMU_S3_Translation主线.md) | 第 94–103 页 | 2026-09-20 按培训模块形成正式文档 | 待测试 |

### 2.7 跨主题综合

跨主题文档不重复解释单个概念，而是用于建立端到端流程和概念之间的联系。

| 文档 | 整理内容 |
|---|---|
| [Device 请求、翻译与软件维护闭环](../03_SYNTHESIS/End_to_End_Flows/01_Device请求_翻译与软件维护闭环.md) | 串联设备请求、Translation Context、地址翻译、缓存、失效和软件同步 |
| [身份、属性与完成语义对照](../03_SYNTHESIS/Concept_Comparisons/01_身份_属性与完成语义对照.md) | 对照请求身份、地址空间、内存属性、事务属性和完成保证 |

第 104 页之后的 Caching、Prefetch、Hazarding、Additional Features、Configuration、Integration 和 Protection Mechanisms 尚未形成系统性的正式知识文档。

## 3. 测试进行到哪里

### 熟练度等级

| 等级 | 代表的能力 | 测试中的典型表现 |
|---|---|---|
| L0 — Seen | 见过这个知识 | 能识别术语，知道它大致出现在哪个章节，但还不能用自己的语言准确解释 |
| L1 — Define | 能说明“它是什么” | 能说出基本定义、主要职责或输入输出；遇到“为什么需要它”“它不负责什么”时仍容易说不清 |
| L2 — Explain | 能说明“为什么以及怎样工作” | 在少量提示下，能解释设计目的、职责边界和基本流程，也能完成简单示例或地址计算；但跨概念连接和独立表达还不稳定 |
| L3 — Connect | 能独立建立完整知识链 | 不看资料也能结构化讲清端到端流程，说明上下游概念之间的关系，并能回答区别、边界、“为什么”和“如果改变条件会怎样”等追问 |
| L4 — Apply | 能将知识用于新问题 | 面对没有练习过的场景，能够进行配置推演、结果预测、故障定位或实际集成分析，并能解释判断依据与方案取舍 |

等级判断遵循以下原则：

- 熟练度必须由实际作答和追问结果支持，不因已经阅读或形成正式知识文档而自动提升。
- 在提示下答对通常只能证明 L2；L3 需要脱稿表达、概念串联和连续追问证据。
- L4 不能仅靠复述知识确认，需要通过陌生场景、配置推演、故障分析或实际问题留下应用证据。
- 表中的等级表示该次测试稳定表现出的最高能力，不自动代表整个大主题都达到同一等级。

本节按照测试发生时间建立索引，不统计知识主题测试覆盖率。

每一行代表一次独立测试；熟练度和问题结论只适用于该次测试，不自动代表整个大主题的掌握水平。

| 日期 | 测试主题 | 测试内容 | 熟练度 | 测试发现的问题 | 完整记录 |
|---|---|---|---|---|---|
| 2026-09-11 | MMU 基础模型首测 | MMU、页表、TLB、数据 Cache、访问权限、ASID、地址计算及完整 load 流程 | L2 已验证，L3 待确认 | 混淆 ASID 与页表入口的作用；条件边界表达不够准确；完整 load 流程尚不稳定 | [查看测试记录](../04_INTERVIEW/Assessments/2026-09-11_MMU基础模型首测.md) |
| 2026-09-14 | Armv9 基础层综合测试 | 多级页表、Translation Regime、Descriptor属性、Stage 1配置与维护、Fault及完整load口述 | L2 稳定，整体L3未确认 | MMU启用配置和地址切换未掌握；BBM/barrier、多级descriptor、VMID及完整口述不稳定 | [查看测试记录](../04_INTERVIEW/Assessments/2026-09-14_Armv9基础层综合测试.md) |

后续每次测试只建立一份 Assessment，并在本节增加对应索引。Assessment 集中保存检验对象、知识点掌握状态、动态拓展说明、原始作答、逐题反馈和复测重点。
