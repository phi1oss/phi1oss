# MMU S3 组件与设备集成拓扑

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 32–33、58–68、74（不是单张 slide 编号）。

## 核心结论与设计目的

SMMU 将地址转换与保护能力扩展到设备访问。MMU S3 的分布式结构把靠近设备的快速查询放在 TBU，把集中式配置/页表查询放在 TCU。判断一种集成方式，首先问“真实 DMA 数据经过哪里”，其次问“谁缓存和使用翻译结果”。

## 1. 各组件的职责

培训第 62–63 页：

| 组件/接口 | 主要职责 |
|---|---|
| TBU | 用本地 TLB 缓存翻译信息 |
| TCU | Configuration/Translation/GPT walks 及集中式翻译服务 |
| BAS interconnect | 在多个 TBU/PCIe Root Port 与 TCU 之间承载 DTI |
| DTI | 分布式翻译和控制消息语义 |
| LTI | 紧耦合组件间的翻译请求通道；响应按协议立即使用，不作为可任意缓存的 ATS 结果 |

ACE-Lite TBU 包含 BIU 与 TLBU；LTI TBU 将翻译服务提供给实现了 LTI 的设备侧组件。CPU/设备搬运的数据不因查询页表而被送进 DTI。

## 2. 三种集成方式

| 方式 | 翻译结果怎么取得 | 真实数据路径 |
|---|---|---|
| Inline | 访问到达后按需要查询 TBU/TCU | 穿过 ACE-Lite TBU |
| Cached | ATS 提前查询，设备 ATC 缓存结果 | 仍经过 SMMU 数据路径；已完成的翻译阶段无需重复 |
| Lookaside | 通过翻译服务获得结果 | 设备/控制器直接向系统互连发数据访问 |

培训第 64–65 页的图中，DTI/BAS 控制路径与 ACE-Lite 数据路径分开。Lookaside 并不意味着系统无需保护；相应 LTI 语义、检查位置及最终数据路径要由集成保证。

## 3. CPU MMU、DMA 与 SMMU

DMA 发起数据传输；SMMU 对相应地址、权限和属性进行处理，而不是代替 DMA 搬运 payload。培训第 33 页的 NVMe 例子说明软件建立映射、设备传输和完成后回收的协作关系，不应把示例驱动调用或简略 unmap 步骤当作跨内核版本固定实现。

## 4. Bifurcation 对拓扑的影响

第 66 页展示宽 PCIe 连接拆分后的多控制器/多接口接入示例。其学习重点是：拆分后的请求来源仍需正确接入 TBU/LTI/DTI-ATS，并保持 SID 与返回路由正确。

讲义不足以证明当前项目的 Root Port 数、BDF→SID 算法、TBU 实例数或任何固定一一对应关系；这些仍是集成待核实项。

## 5. 流程定位与自测

```text
设备访问 → 数据接口 / 本地翻译服务
                    ↓ cache miss
              DTI → BAS → TCU → 配置/页表查找
                    ↑
                 返回翻译信息
数据路径根据结果继续执行，并遵守适用检查
```

自测：在 Inline 和 Lookaside 图上分别指出 payload、DTI 与 LTI 的位置。为什么“LTI 响应立即使用”不否认 LTI TBU 内部存在 TLB？

一句话：TBU 管设备附近的翻译缓存，TCU 管集中查询；集成方式决定数据走哪里，DTI/BAS 连接翻译服务。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 32–33、58–68、74 页；文内“培训依据”均指上述页面。
- 来源问答：[01_TBU_TCU_DTI与DTI-ATS概述](../../01_INBOX/ChatGPT/01_TBU_TCU_DTI与DTI-ATS概述.md)
- 来源问答：[39_SMMU三种设备集成方式](../../01_INBOX/ChatGPT/39_SMMU三种设备集成方式.md)
- 来源问答：[43_MMU_S3系统定位与能力分层](../../01_INBOX/ChatGPT/43_MMU_S3系统定位与能力分层.md)
- 来源问答：[44_PCIe_Root_Complex_Root_Port与Bifurcation](../../01_INBOX/ChatGPT/44_PCIe_Root_Complex_Root_Port与Bifurcation.md)
- 来源问答：[45_DMA机制_SoC应用与SMMU隔离](../../01_INBOX/ChatGPT/45_DMA机制_SoC应用与SMMU隔离.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
