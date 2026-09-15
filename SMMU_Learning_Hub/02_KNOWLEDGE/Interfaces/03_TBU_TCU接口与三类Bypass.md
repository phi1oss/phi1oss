# TBU/TCU 接口与三类 Bypass

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 41、71、73–76、79（不是单张 slide 编号）。

## 核心结论与设计目的

接口名称必须与承载的流量对应：TBS/TBM 是设备数据通路，DTI 是翻译控制通路，QTW/PTW 是 TCU 发起的内存访问，PROG 是软件寄存器入口。Bypass 则必须区分全局、Stream 和每事务物理旁路，不能统称“不做任何处理”。

## 1. 接口职责

| 接口 | 主要用途 | 边界 |
|---|---|---|
| TBS / TBM | ACE-Lite TBU 的输入/输出数据接口 | 处理设备 transaction，非 DTI payload |
| LTI | 与紧耦合设备侧翻译使用者通信 | 翻译请求及结果使用受 LTI 协议约束 |
| DTI / BAS | TBU、Root Port 与 TCU 的翻译控制消息 | 不搬运 DMA 数据 |
| QTW/DVM | 配置/表访问、队列、DVM、适用 MSI 等 | 是否承载 PTW 取决于 TCU 变体 |
| 可选 PTW | 专用页表遍历及 HTTU 路径 | x2 变体的带宽分流 |
| PROG / APB5 | 配置和访问寄存器 | 访问 TBU 特定寄存器时可经 DTI 代理 |
| MSI | TCU 通知路径 | 专用 BAS 或适用 QTW 路径依集成选择 |

## 2. QTW/PTW 的变体边界

培训第 75 页先列总流量，再列可选专用 PTW。TRM §2.4.1.1–2 明确：tcu_x2 存在专用 PTW，PTW/HTTU 等指定流量不再走 QTW/DVM；单口场景由 QTW/DVM 承担相应访问。

TRM 列出的 QTW transaction types 为 ReadNoSnoop、WriteNoSnoop、ReadOnce、WriteUnique、DVM Complete，以及仅单口时的 AtomicCompare；专用 PTW 列 ReadNoSnoop、ReadOnce、AtomicCompare。

HTTU 是硬件对翻译表状态的更新，不是 DMA payload 写，也不是软件 CFGI 的另一种名称。TRM 此段虽提及 DPT traffic，但能力表确认 MMU-720AE 不支持 DPT；不能由流量列表反推功能已启用。

## 3. 三类 Bypass

| 类型 | 选择位置 | 地址与属性 |
|---|---|---|
| Global Bypass | 全局使能与 GBPA 等控制 | 依配置旁路或 Abort，适用属性控制 |
| Stream Bypass | STE / Stream 配置 | 不做相应地址翻译，但仍可转换属性 |
| NoStreamID / Physical Bypass | AxMMUVALID=0；LTI 有对应 LAMMUV | 输入必须是 PA，旁路架构翻译与属性转换，培训说明仍执行 GPC |

培训第 41 页“关闭 SMMU 时都 Bypass”是简化描述。架构第 109–110 页指出还要看 GBPA.ABORT 等控制，不能推出 disable 等于无条件放行。

第 73 页：AxMMUVALID=1 表示可进入翻译流程，而非保证每笔都执行页表 walk；为 0 时其他 AxMMU* 信息无效。它适用于受信任访问路径，不是给任意设备绕过隔离的开关。

## 4. 寄存器代理访问

软件从 TCU PROG 访问 TBU-specific register → TCU 发 DTI_TBU_REG_READ/WRITE → TBU 返回 RDATA/WACK。这是寄存器管理，不是用 REG_READ 读取 STE/CD/PTE。

## 5. 待核实与自测

当前项目采用 x1/x2、哪些端口实际接线、MSI 目标及 NoStreamID 信任控制仍需设计资料。精确 AxID 和信号位宽不在本篇推断。

自测：关闭 SMMU 一定会放行 DMA 吗？Stream Bypass 与 AxMMUVALID=0 的属性行为有什么不同？

一句话：先区分数据、翻译控制和表访问路径，再判断 Bypass 的作用层级与保护边界。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 41、71、73–76、79 页；文内“培训依据”均指上述页面。
- 产品依据：[MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)，第 32–36、79–81 页。
- 架构依据：[SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)，第 109–110 页，全局 Bypass/Abort。
- 来源问答：[07_QTW_DVM的六种Transaction_Type](../../01_INBOX/ChatGPT/07_QTW_DVM的六种Transaction_Type.md)
- 来源问答：[08_PTW_HTTU_DPT与QTW分流](../../01_INBOX/ChatGPT/08_PTW_HTTU_DPT与QTW分流.md)
- 来源问答：[22_SMMU_Bypass模式](../../01_INBOX/ChatGPT/22_SMMU_Bypass模式.md)
- 来源问答：[26_SMMUv3寄存器空间与Memory-based配置](../../01_INBOX/ChatGPT/26_SMMUv3寄存器空间与Memory-based配置.md)
- 来源问答：[48_ACE5-Lite接口Properties与事务语义](../../01_INBOX/ChatGPT/48_ACE5-Lite接口Properties与事务语义.md)
- 来源问答：[58_DTI远程访问TBU寄存器](../../01_INBOX/ChatGPT/58_DTI远程访问TBU寄存器.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
