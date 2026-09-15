# Configuration Lookup 与 Translation Lookup

> 记录日期：2026-09-01  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 原始上下文：培训视频截图，时间点未知  
> 状态：Inbox 问答整理，待结合 SMMUv3 Architecture Specification 与 MMU-720AE TRM 校验蒸馏

## 凝练问题

**如何理解培训图中 SMMU 从接收一笔 Transaction，到通过 SID/SSID 确定 Translation Context，再查 TLB 或执行 Stage 1/Stage 2 Table Walk，最终形成 Output Transaction 的完整流程？**

## 核心结论

> **SMMU 的 Translation 可以分成两个逻辑阶段：Configuration Lookup 先用 StreamID/SSID 找到这笔 Transaction 应使用的 Translation Context；Translation Lookup 再结合该 Context、输入地址和访问属性查找 Cached Translation，命中时直接得到输出地址与权限，未命中时按相应 Stage 1/Stage 2 配置执行 Table Walk。**

一句话记忆：

> **Configuration Lookup 回答“用哪套翻译环境”；Translation Lookup 回答“这个地址最终翻到哪里、是否允许访问”。**

## 1. Incoming Transaction 提供什么

一笔进入 SMMU 的 Transaction 概念上携带：

```text
StreamID
SubstreamID / SSID（如果使用）
Input Address
访问类型与属性
Data Payload（读写 Transaction 中按协议存在）
```

其中 SID/SSID 用于选择 Translation Context，Input Address 与访问属性用于执行地址翻译和权限检查。

## 2. 第一阶段：Configuration Lookup

Configuration Lookup 回答：

> **“这笔 Transaction 应该使用哪套 Translation Configuration？”**

首先由 StreamID 选择 STE：

```text
StreamID
   ↓
Stream Table Lookup
   ↓
STE
```

STE 提供 Stream 级别的信息，例如：

```text
Stream/Translation World
Translation Mode
VMID
Stage 2 Translation-table Information
S1ContextPtr
其他 Stream Configuration
```

SID 因此确定该请求属于哪个 Stream，以及该 Stream 采用什么 Stage 1/Stage 2 Translation Environment。

## 3. 使用 Substream 时继续查找 CD

如果该 Stream 启用 Substream/SSID：

```text
STE.S1ContextPtr
      ↓
Context Descriptor Table
      ↑
     SSID
      ↓
      CD
```

CD 提供 Stage 1 Translation Context，例如：

```text
ASID
Stage 1 Translation-table Base
Stage 1 Translation Control
Memory Attribute Configuration
其他 Stage 1 参数
```

前半段可压缩为：

```text
SID  → STE → Stream-level Configuration / Stage 2 Context
SSID → CD  → Substream-level Stage 1 Context
```

如果不使用 Substream，Stage 1 Context 的选择方式和默认 SSID/单 CD 行为应按 STE 配置及架构规则解释，不能一概认为每笔请求都必须显式携带 SSID。

## 4. Configuration Lookup 的输出

Configuration Lookup 的结果不是 PA，而是一套用于解释地址的完整 Translation Context。培训图概念上包括：

```text
Security / PAS-related Context
StreamWorld / Translation Regime
ASID
VMID
Stage 1 Table Information
Stage 2 Table Information
Input Address 与访问属性
```

因此它的本质是：

> **把 SID/SSID 解析成“这笔地址应该在哪个安全状态、Stage 1 地址空间和 Stage 2 虚拟化上下文中解释”。**

可以与 CPU 侧作有限类比：

```text
CPU：当前 EL/Security + TTBR/TCR + ASID/VMID
     → 当前 Translation Regime/Context

SMMU：SID/SSID + STE/CD
      → 当前 Transaction 的 Translation Context
```

CPU Context 主要由当前执行状态和系统寄存器决定；SMMU 可能并发服务多个 Stream，因此按每笔 Transaction 的 SID/SSID 选择 Context。

## 5. 第二阶段：Translation Lookup

获得完整 Context 后，SMMU 才开始回答：

> **“这个 Input Address 对应哪个 Output Address，访问是否满足权限？”**

第一步通常是查找 Translation Cache/TLB：

```text
Translation Context
+ Input Address
+ Access Attributes
        ↓
Translation Cache / TLB Lookup
```

培训图强调：

> **Translation Lookup 不是只按 Address 匹配，它必须在正确的 Context 下匹配。**

同一个地址 `0x4000` 可能同时存在于：

```text
不同 ASID
不同 VMID
不同 Security/PAS Context
不同 StreamWorld/Translation Regime
```

这些 Context 下的 Translation Result 可以完全不同。

## 6. 如何理解 TLB 的 Context-sensitive Tag

概念上可以把 Cached Translation 理解为：

```text
{Security/PAS Context,
 StreamWorld/Regime,
 ASID,
 VMID,
 Input Address}
        ↓
Output Address + Permissions/Attributes
```

这表示 Translation Cache 不能跨不兼容的地址空间或安全上下文误用结果。

需要注意边界：

> **上式是帮助理解架构匹配语义的概念模型，不代表所有 SMMU 实现都必须把这些字段按同样形式直接拼接成一组物理 Tag Bit。具体 TLB 组织、分级缓存和 Tag 编码属于实现细节。**

## 7. TLB Hit 路径

如果 Translation Lookup 命中：

```text
Context + Input Address
          ↓
        TLB Hit
          ↓
Output Address
+ Permissions/Attributes
```

此时不需要再次读取 Stage 1/Stage 2 Translation Table，可以直接使用缓存的 Translation Result，同时仍需按架构规则检查该 Transaction 的访问类型和权限。

TLB 的核心性能价值是：

> **避免每笔 Transaction 都执行多级 Table Walk。**

## 8. TLB Miss 与 Table Walk

如果未命中：

```text
TLB Miss
   ↓
使用 Configuration Lookup 得到的
S1/S2 Table Base 与控制信息
   ↓
Walk Translation Tables
```

双阶段 Translation 的概念路径为：

```text
Input IOVA/VA
      ↓
Stage 1 Table Walk
      ↓
IPA
      ↓
Stage 2 Table Walk
      ↓
PA
```

完成后得到：

```text
Output Address
Permissions
Memory/Transaction Attributes
```

成功结果通常会被填入 Translation Cache/TLB，以便后续相同 Context 和地址的访问直接命中：

```text
Miss → Walk → Translation Result → Fill Cache/TLB
```

如果配置无效、页表项无效、权限不允许或其他检查失败，则会按相应 Fault Model 产生 Abort、Stall/Event 等结果，而不是形成正常 Output Transaction。

## 9. ASID 与 VMID 为什么都参与 Context 区分

### ASID

用于标识 Stage 1 Address Space：

```text
Process A：ASID = 10
Process B：ASID = 20
```

即使两个进程使用相同 VA，它们的 Stage 1 Translation 也不会混淆。

### VMID

用于标识 Stage 2 Virtualization Context：

```text
VM1：VMID = 3
VM2：VMID = 7
```

即使两个 VM 出现相同 IPA，其 Stage 2 Translation Result 仍可以不同。

可以记为：

```text
ASID → 区分 Stage 1 Address Space
VMID → 区分 Stage 2/VM Address Space
```

## 10. StreamWorld/Security Context 的作用

培训图把 StreamWorld 和 Security 一并送入 Translation Lookup，想强调：

> **Cached Translation 必须属于正确的 Architectural Transaction Regime 与安全上下文，不能只凭 Address、ASID 和 VMID 就跨 World 或跨安全域复用。**

StreamWorld、Security/PAS 与具体 TLB 匹配和缓存维护之间的精确架构规则，需要按所使用的 SMMUv3 版本、Security Extension/RME 能力和具体实现继续校验。

## 11. Data Payload 与 Translation Logic 的关系

培训图底部将 Data 从 Input Transaction 连到 Output Transaction，表达的是：

> **Configuration Lookup 和 Translation Lookup 主要处理 Transaction 的地址、上下文、权限和相关属性，Table Walk 不是对 DMA Data Payload 本身做转换。**

概念上：

```text
Input Address + Context/Attributes
              ↓
      SMMU Translation Logic
              ↓
Output Address + Checked Attributes

Data Payload
      +
Translated/Checked Transaction Information
              ↓
       Output Transaction
```

图中的旁路画法不应推导成 Data Payload 在微架构中一定零延迟或完全不需要 Buffer；Payload 如何暂存、与翻译结果如何重新组合属于具体实现和总线协议行为。

## 12. 端到端流程

```text
Incoming Transaction
   │
   ├─ StreamID
   ├─ SubstreamID（可选/按配置）
   ├─ Input Address
   └─ Access Attributes
          ↓
   Configuration Lookup
          ↓
      SID → STE
          ↓
      SSID → CD（需要时）
          ↓
得到：
Security / StreamWorld
ASID / VMID
S1/S2 Table Configuration
          ↓
   Translation Lookup
          ↓
   Translation Cache/TLB
      /             \
    Hit             Miss
     │                ↓
     │          Walk S1/S2 Tables
     │                ↓
     └────────→ Output Address
                  + Permissions
                        ↓
               Output Transaction
```

## 面试式回答

> **SMMUv3 的 Translation 可以逻辑上分为 Configuration Lookup 和 Translation Lookup 两阶段。首先通过 StreamID 找 STE，必要时再用 SubstreamID 找 CD，从而获得该 Transaction 的 Security/StreamWorld、ASID、VMID 及 Stage 1/Stage 2 Translation-table Configuration；然后用这些 Context 信息、输入地址和访问属性查找 Translation Cache。命中时直接得到输出地址和权限，未命中时根据对应的 Stage 1/Stage 2 配置执行 Table Walk，并通常将成功结果缓存。**

## 一句话总结

> **先用 SID/SSID 回答“该用谁的页表和地址空间”，再用 Context+Address 回答“地址翻到哪里、权限是否允许”。**

## 相关记录与资料

- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [StreamID 与 Translation Context 映射](18_StreamID与Translation_Context映射.md)
- [Substream 如何关联进程地址空间](28_Substream如何关联进程地址空间.md)
- [SMMU Fault Model：Abort、Stall 与 Resume](21_SMMU_Fault_Model_Abort_Stall与Resume.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

