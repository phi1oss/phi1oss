# PCIe ATS 两种模式与 ATC 维护

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 50、63–65、89（不是单张 slide 编号）。

## 核心结论与设计目的

ATS 允许设备预先请求翻译并把结果缓存在 ATC。Full 与 Split-stage 的关键区别是：设备取得的是最终 PA，还是仍需 Stage 2 的 IPA。是否携带 Translated 标记，不能单独说明已完成哪些阶段，也不是安全认证。

## 1. 翻译请求与真实 DMA 分开

```text
翻译查询：Endpoint → PCIe Root Port → DTI-ATS → TCU → 返回结果 → ATC
数据访问：Endpoint 使用缓存地址 → 控制器/数据路径 → 系统内存
```

DTI-ATS 承载请求、结果与维护信息，不承载 DMA payload。ATC 只缓存有效结果，不负责创建 STE/CD 或软件页表。

## 2. Full / Split-stage 对比

按培训第 64 页两阶段示例：

| 维度 | Full ATS | Split-stage ATS |
|---|---|---|
| ATC 缓存结果 | S1+S2 的最终结果 | S1 的结果 |
| 设备后续地址 | PA | IPA |
| 后续 Stage 2 | 不重复已完成的翻译 | SMMU 仍执行 |
| 安全侧重点 | 需要信任和适用的物理访问保护 | 保留最终 Stage 2 边界 |
| 培训列出的限制 | GPC 不能自动隔离同 PAS 内所有 Realm | Fully coherent 与 PCIe P2P 受剩余 Stage 2 约束 |

这里 Full 的示例假定 S1/S2 都启用；它的本质是完成该配置要求的全部翻译，并非所有 Full ATS 场景都必有两级。

## 3. Translated、EATS 与配置查找

原问答提出“Endpoint 如何知道 IPA/PA”。应在 SMMU Stream 配置与输入 transaction 类型的组合下解释地址，不能仅看数值，也不能臆造一个每包都有的 Full/Split 标志。

STE.EATS 是相关模式控制之一。其精确取值、与其他 STE 字段和 transaction type 的合法组合尚未在本轮逐项查证，正文只晋升模式语义，不给可直接使用的寄存器编码。

即使不再执行完整 translation walk，也不能推出无需配置查找、权限判断或 GPC。

## 4. 映射更新需要考虑 ATC

页表或 Context 更新后，旧结果可能同时存在 SMMU 缓存与设备 ATC。维护范围须包含适用的 TLBI/CFGI、ATC invalidation 和同步，不能认为清掉 TBU TLB 就消除了所有设备侧旧结果。

规范中的 CMD_SYNC 对此前 CMD_ATC_INV 有完成要求，但异常/超时不等于成功，见维护文档。

## 5. 安全推导的边界

培训明确指出 Split-stage 的功能限制；“设备只拿到 IPA，因此不能直接参与 PA 视角的某些流程”是帮助理解的推导，不是对所有未来系统的通用不可能性证明。准确的 Fully coherent/P2P/ACS 条件仍需对应互连与 PCIe 规范。

DPT 只作为培训 MMU S3 r1 的安全演进背景，不能作为 MMU-720AE 已实现能力。

自测：Full ATS 中为什么仍有安全检查？Split-stage 为什么不能用 ATC 中的结果直接当最终 PA？

一句话：Full 把最终翻译结果交给设备，Split-stage 把最终 Stage 2 留在 SMMU；两者都需要管理缓存与安全边界。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 50、63–65、89 页；文内“培训依据”均指上述页面。
- 架构补充：[SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)，§4.7.3 第 241–243 页，ATC invalidation 完成边界。
- 来源问答：[35_PCIe_ATS与ATC工作流程](../../01_INBOX/ChatGPT/35_PCIe_ATS与ATC工作流程.md)
- 来源问答：[36_Full_ATS的安全边界](../../01_INBOX/ChatGPT/36_Full_ATS的安全边界.md)
- 来源问答：[39_SMMU三种设备集成方式](../../01_INBOX/ChatGPT/39_SMMU三种设备集成方式.md)
- 来源问答：[40_Full_ATS与Split-stage_ATS](../../01_INBOX/ChatGPT/40_Full_ATS与Split-stage_ATS.md)
- 来源问答：[41_DPT为Full_ATS补充设备权限检查](../../01_INBOX/ChatGPT/41_DPT为Full_ATS补充设备权限检查.md)
- 来源问答：[42_Split-stage_ATS的功能限制](../../01_INBOX/ChatGPT/42_Split-stage_ATS的功能限制.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
