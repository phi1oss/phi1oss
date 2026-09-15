# Secure MSI 与 ITS 路径的架构边界及项目实现

> 记录日期：2026-09-02  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`，对 Global Error slide 的追问  
> 类型：待核实问题  
> 状态：架构层已有初步判断；MMU-720AE + GIC-600AE 项目集成路径尚未确认

## 问题

**如果 MSI 通常发送给 ITS，而 ITS/LPI 路径只能形成 Non-secure 中断，那么 Secure 场景下的 message-based interrupt 应如何处理？“MSI”本身是否存在 Secure 与 Non-secure 两种类型？在当前 MMU-720AE + GIC-600AE 项目中，SMMU 的 Secure interrupt 实际走哪条路径？**

## 当前初步认识

以下内容可作为后续查证的工作假设，暂不视为项目实现结论：

1. **MSI 的机制本质是一笔 memory write。** 不宜笼统理解为协议中存在一个通用的 `MSI.type = Secure/Non-secure` 字段；进入 Arm 系统后的 transaction security/PAS 属性、目标 GIC message interface 和 INTID 配置需要分别判断。
2. **标准 ITS 路径最终产生 LPI，而 LPI 属于 Non-secure Group 1。** 因此不能依赖 `ITS → LPI` 路径产生 Group 0 或 Secure Group 1 interrupt。
3. **Secure message-based interrupt 需要走支持 Secure interrupt 的其他路径。** 可能包括 GIC 的 Secure message-based SPI 接口（例如与 `GICD_SETSPI_SR` 相关的路径）或 wired Secure SPI；具体选择取决于 SoC 集成。
4. **PCIe MSI 与 Arm TrustZone Security 属性不能直接画等号。** PCIe Memory Write 进入 Arm SoC 后如何映射为系统安全属性，需要结合 Root Complex、interconnect、SMMU 和 GIC 集成确认。

## 尚未解决的关键点

- MMU-720AE 的 Secure/Non-secure interrupt 输出分别支持 wired、MSI 还是两者可选。
- `SMMU_S_*` 相关中断的 MSI address、data 和 transaction security attribute 如何配置。
- GIC-600AE 在当前系统中是否启用了 Secure message-based SPI，地址窗口如何映射，访问权限如何限制。
- 当前项目是否把 Secure SMMU interrupt 接为 wired SPI，而只让 Non-secure interrupt 使用 MSI/ITS。
- 当报告的 Global Error 本身是 MSI write abort 时，系统是否提供独立且可靠的 fallback 通知路径。

## 后续核查资料与顺序

1. 查 GICv3/GIC-600AE 文档，确认 LPI 的 Security Group 限制及 Secure message-based SPI 的精确定义。
2. 查 MMU-720AE TRM 的 interrupt、MSI、Secure register frame 和 integration signal 章节。
3. 查 SoC 顶层地址映射与连接图，确认 SMMU 的 MSI target address 是否指向 ITS、Distributor message register 或其他 interrupt interface。
4. 查当前项目 firmware/driver 对 `SMMU_(S_)IRQ_CTRL`、MSI 配置寄存器和 GIC 路由的初始化代码。
5. 最终分别形成“架构允许什么”和“当前项目实际怎么接”两项结论，避免用通用 GIC 机制替代项目事实。

## 预期输出

- 一张 `SMMU interrupt source → MSI/wired path → GIC target → Security Group` 的项目级路径表。
- 明确“Secure MSI”在本项目语境中究竟指 Secure transaction、Secure message-based SPI，还是只是口语化称呼。

## 关联记录

- [Inbox 34：EVTQ 与 Global Error 的职责边界](../ChatGPT/34_EVTQ与Global_Error的职责边界.md)
- [Inbox 16：SMMU TrustZone 与 Security State 机制](../ChatGPT/16_SMMU_TrustZone与Security_State机制.md)

