# PMCG 与 PMU Snapshot

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 80（不是单张 slide 编号）。

## 核心结论与设计目的

每个 TBU 和 TCU 各有 PMCG，用于统计架构及实现事件。实时计数器持续变化；Snapshot 将同一组计数器的值捕获到 shadow registers，便于软件读取一致的组内快照。

## 1. 三类寄存器必须分清

| 对象 | 用途 |
|---|---|
| SMMU_PMCG_EVCNTRn | 读取实时计数值 |
| SMMU_PMCG_SVRn | 读取捕获后的影子值 |
| SMMU_PMCG_CFGR.CAPTURE | 只读能力声明：是否支持 capture |
| SMMU_PMCG_CAPR.CAPTURE | 写 1 触发软件 capture |

培训第 80 页和 TRM 第 325 页都出现了“写 CFGR.CAPTURE”的说法；TRM §2.5.2.5 第 56 页及架构 §10.5.2.11/CFGR 字段说明明确区分 CAPR 与 CFGR。本篇按后两者校正，原问答 51 保留作为来源。

## 2. 硬件快照握手

培训第 80 页和 TRM 第 56 页：pmusnapshot_req/ack 使用四阶段异步握手。复位后两者为低，request 触发 capture，ack 确认，再撤 request、撤 ack 回到初始状态。

这不是软件在多个地址上快速轮询的同义词；快照保存在 SVRn 中，真实计数可以继续进行。

## 3. 组内同时，不保证全芯片绝对同时

产品描述保证同一 PMCG 的 counter snapshot。多 TBU/TCU 可以由 CTI 等调试基础设施协调触发，但跨 clock domain 的采样延迟和触发分发不由“snapshot”一词自动消除。

因此不能仅凭所有组件都接了请求信号，就声称它们在同一个物理时刻捕获。跨组对齐精度需要系统设计或测量证据。

## 4. 用计数器判断瓶颈

先选事件及适用 SID filter → 明确统计窗口 → 捕获/读取 → 比较增量与分母 → 检查计数溢出和事件定义。

TRM 第 47–48 页中，TBU 和 TCU 的相同编号事件未必代表完全相同的统计对象，例如 TBU 的 miss 指标与发往 TCU 的请求相关，TCU 则可按实际 walk 行为计数。不能不看事件定义就相除得到所谓“全系统 TLB miss rate”。

产品第 56 页声明 PMCG 不支持 MSI；不要把 TCU 其他中断支持 MSI 推广到 PMCG。

## 5. 待验证与自测

Counter 数量、当前事件配置和实际 workload 尚未知；本篇不填写测量值或性能结论。跨组精确同步、溢出处理与计数宽度还需按当前配置验证。

自测：CFGR.CAPTURE 与 CAPR.CAPTURE 有何区别？两组 PMU 同时收到外部请求是否就能保证同周期快照？

一句话：EVCNTR 看实时，SVRn 看快照，CFGR 查能力，CAPR 发触发；组内一致不等于全芯片绝对同时。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 80 页；文内“培训依据”均指上述页面。
- 产品依据：[MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)，第 47–48、56 页（与第 325 页存在字段名称冲突）。
- 架构依据：[SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)，第 993、1023、1028 页，CAPR 与 CFGR。
- 来源问答：[51_SMMU_PMCG与PMU_Snapshot](../../01_INBOX/ChatGPT/51_SMMU_PMCG与PMU_Snapshot.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
