# AxCACHE、AxDOMAIN 与属性转换边界

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 16–18、71–76（不是单张 slide 编号）。

## 核心结论与设计目的

缓存策略与共享范围是两个概念维度，但落到总线信号时，合法组合、事务类型及产品归一化规则会一起约束最终行为。因此“AxCACHE 管缓存，AxDOMAIN 管共享范围”适合作为入口，不能替代协议或 MMU-720AE 的转换表。

## 1. 从页表属性到总线属性

培训第 16–18 页：

- Stage 1 PTE.AttrIndx 选择 MAIR 属性项，得到 Normal/Device 及适用的 Inner/Outer cache policy。
- PTE.SH 描述适用的 Shareability；不存放在 MAIR 中。
- Inner/Outer Cacheability 不固定等同于 L1/L2/L3；Shareability 的 Inner/Outer 也不是缓存层级编号。
- CD 的 IR/OR/SH 是 table-walk 访问属性，与最终数据页属性分开。

总线侧再将解析后的架构属性转换为 AxCACHE、AxDOMAIN 等信息。这个过程可能不是无损一一映射。

## 2. 两个信号的适用范围

AxCACHE 承载 memory/cache 行为的总线编码；ARCACHE 与 AWCACHE 的同一 bit 位置不能脱离读/写编码规则随意解释。

AxDOMAIN 是 ACE/ACE-Lite 的域相关属性，不是基础 AXI 所有版本的必有信号。Non-shareable、Inner/Outer、System 等总线取值也不能逐字等同于 PTE.SH 的全部编码。

Non-cacheable 不意味着不可能与已缓存副本保持一致；Non-shareable 更不等于其他 agent 被禁止访问。是否 snoop、是否真正 coherent，还要看 AxSNOOP、事务类型、互连能力与集成。

## 3. MMU-720AE 并非逐字段透传

TRM §2.5.11 与 §2.6.2.4 描述 TBS 输入和 TBM/QTW/PTW 输出转换：

- Write-Through 输入在表 2-19 中被归一化成 Normal-iNC-oNC。
- Normal-iWB-oWB 输出可表达为 Write-Back；其他 Inner/Outer 策略组合可能被归一化为 Non-cacheable。
- TBM 的 AxUSER Outer Cacheable 位补充转换前的 Outer Cacheability 信息。
- 不能认为“PTE.SH=ISH”必然在下游得到同名 AxDOMAIN 编码。

这也是“属性独立”不等于“信号能任意组合”的实例。

## 4. 已发现的资料内部差异

TRM 第 73 页表 2-14 的 Write-Back Shareability 行与第 86 页表 2-19 不一致：后者明确写所有 ISH 转 OSH，前者抽取及表项呈现为另一映射。第 74 页又明确说明输出 ISH 转 OSH。

本篇仅确认存在转换/归一化及其相关表述，不把冲突的输入 ISH 映射当成已解决结论。精确 TBS 行为需对照产品勘误、集成说明或实现确认。未因讲义/旧回答更容易记忆而忽略冲突。

## 5. 自测与面试一句话

自测：设置 AxCACHE=Cacheable 是否足以保证 coherent DMA？为什么页表 Inner WB / Outer NC 未必能原样保留到一个 AxCACHE 字段？Non-shareable 能否充当访问权限？

一句话：架构属性定义缓存行为与共享范围，总线编码负责表达；MMU-720AE 还会按产品规则转换，不能只凭字段名猜行为。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 16–18、71–76 页；文内“培训依据”均指上述页面。
- 产品依据：[MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)，§2.5.11 第 72–74 页、§2.6.2.4 表 2-19/2-20/2-21 第 86–88 页。
- 来源问答：[49_ACE5-Lite系统级Properties与翻译输出属性](../../01_INBOX/ChatGPT/49_ACE5-Lite系统级Properties与翻译输出属性.md)
- 来源问答：[60_Shareability_Cacheability与AttrIndx](../../01_INBOX/ChatGPT/60_Shareability_Cacheability与AttrIndx.md)
- 来源问答：[62_AxCACHE与AxDOMAIN的含义](../../01_INBOX/ChatGPT/62_AxCACHE与AxDOMAIN的含义.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
