# ACE5-Lite Properties 与输出属性

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 71–76（不是单张 slide 编号）。

## 核心结论与设计目的

Property 是接口支持的能力或协议约定；Signal 是传递信息的载体；某笔 transaction 的字段值才表达它正在使用哪种操作。支持一项 Property 不等于所有事务都启用它，也不等于一定新增信号。

## 1. 保持原事务语义

培训第 71–72 页给出以下能力：

| 类别 | 讲义信号/编码 | 学习重点 |
|---|---|---|
| Cache Stash | AWSNOOP、AWSTASHNID/LPID 及有效位 | 请求将数据放到合适 cache 位置，不是 SMMU 直接写 CPU L1 |
| Atomic | AWATOP | 表达原子操作类型；TBU 翻译地址不等于执行原子运算 |
| Loopback | AxLOOP | 传递事务跟踪信息 |
| Poison | xPOISON | 标记传输数据损坏 |
| Unique ID | AxIDUNQ/xIDUNQ | 当前没有其他在途事务使用相同 AXI ID 的声明 |
| Read Data Chunking | ARCHUNKEN、RCHUNK* | 分块返回所需的信息 |
| CMO / DeAllocation | 复用既有编码 | 没有新信号也可以形成能力差异 |
| Wakeup | AWAKEUP 等 | 为有活动的接口提供唤醒信息 |

“Unique”是受协议条件约束的在途声明，不是永久全系统唯一编号。Loopback、AXI ID、BAS TID 也不应混称同一个 ID。

Stash 目标 Node ID / LP ID 的值与有效位需要系统拓扑支持；仅凭 sideband 不能保证最终放进哪个物理 cache way 或必定驻留。

## 2. 翻译输出不只是 PA

| 属性类别 | 培训信息 | 不负责什么 |
|---|---|---|
| MPAM | TBM AxMPAM，资源分区/监控信息 | 不是地址映射本身 |
| MTE Basic | TAGOP、TAG、WTAGUPDATE | 接口支持不等于 TBU 执行全部 CPU tag 检查 |
| PBHA | AxPBHA，页表相关硬件属性 | 具体解释不是通用固定策略 |
| RME | AxNSE 与 AxPROT[1] 共同表达 PAS | 不是单凭信号验证所有授权 |
| MEC | AxMECID | 不是 VMID 或页表根 |
| InvalidateHint | AWSNOOP 扩展操作 | 不可直接替代所有正确性所需维护 |

表中信息以培训第 72 页为依据。关于应用动机的解释是概念归纳，不为实现额外规定算法。

## 3. TBU 与 TCU 的角色不同

TBU 需要在翻译地址时保留、转换或生成适用属性；TCU 的表访问也有自身属性。培训第 76 页为 QTW/DVM/PTW 单独列出能力，不能把 TBU 全部 Property 复制到 TCU，或反过来。

例如 TRM 第 33 页明确 TCU 该接口不发 exclusive accesses；这不代表设备端 Atomic 与 Exclusive 可混为同一种协议。

## 4. 本轮晋升边界

已晋升：能力/信号/事务值的区分，以及培训明确列出的信号与用途。

仍留来源层：AWATOP 精确分类及 response/operand 规则、Exclusive 完整时序、Read Chunking 所有合法排列、Stash enable 组合及具体 cache placement、Persist 的持久化域。当前库未提供完整相应 AMBA 规范，不将这些细节写成已验证规则。

## 自测与面试一句话

自测：为什么一项 Property 可以没有新信号？为什么有 MECID 仍不能替代 GPC？Atomic 为什么不能用普通读和普通写随意拆分？

一句话：Property 定义接口会什么，信号说明事务做什么；SMMU 翻译地址时还必须保持相应事务语义。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 71–76 页；文内“培训依据”均指上述页面。
- 产品补充：[MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)，第 33 页，TCU 接口事务限制。
- 来源问答：[47_ACE-Lite_Cache_Stash与目标ID](../../01_INBOX/ChatGPT/47_ACE-Lite_Cache_Stash与目标ID.md)
- 来源问答：[48_ACE5-Lite接口Properties与事务语义](../../01_INBOX/ChatGPT/48_ACE5-Lite接口Properties与事务语义.md)
- 来源问答：[49_ACE5-Lite系统级Properties与翻译输出属性](../../01_INBOX/ChatGPT/49_ACE5-Lite系统级Properties与翻译输出属性.md)
- 来源问答：[50_AMBA_Atomic_Transaction与Exclusive_Access](../../01_INBOX/ChatGPT/50_AMBA_Atomic_Transaction与Exclusive_Access.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
