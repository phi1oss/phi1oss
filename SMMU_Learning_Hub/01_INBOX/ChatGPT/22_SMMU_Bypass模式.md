# SMMU Bypass 模式

> 记录日期：2026-08-30  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 来源材料：培训截图，时间点未知  
> 状态：Inbox 问答整理，待结合 SMMUv3 与 MMU-720AE 规范蒸馏

## 凝练问题

**SMMU 的 Bypass 是什么？单个 Stream Bypass 与整个 SMMU Disabled 有什么区别？Stage 1/Stage 2 如何独立选择 Bypass，地址不翻译时 SMMU 是否仍会处理 transaction attributes？**

## 核心结论

> **SMMU Bypass 表示不执行正常的地址翻译，输入地址直接作为下游地址使用；但 transaction 仍然经过 SMMU，因此 SMMU 仍可能覆盖 cacheability、shareability 等 attributes。需要区分 SMMU Enable 时的 Per-Stream Bypass，以及 SMMU Disabled 时所有 transaction 使用的 Global Bypass。**

## 1. Bypass 的基本含义

正常 Translation：

```text
Input Address
    ↓
SMMU
    ↓
Stage 1 / Stage 2
    ↓
Output PA
```

Bypass：

```text
Input Address
    ↓
SMMU
    ↓
不执行地址 Translation
    ↓
Output Address
```

因此最重要的边界是：

> **Bypass 是绕过地址翻译，不是让 transaction 绕开 SMMU 硬件，也不等于关闭 SMMU 的所有处理。**

## 2. Per-Stream Bypass

SMMU 已经 Enable 时，可以根据 Stream 的配置分别选择 Translate 或 Bypass：

```text
SMMU = Enabled

Stream A → Translate
Stream B → Bypass
Stream C → Translate
```

例如：

```text
DMA0
SID           = 10
Input Address = 0x8000_0000

STE 配置为 Bypass
        ↓
SMMU 不读取正常 Translation Table
        ↓
Output Address = 0x8000_0000
```

也就是说，同一个 SMMU 可以同时服务 Bypass Stream 和 Translated Stream。

## 3. 为什么 Bypass 仍能修改 Attributes

即使地址不变，transaction 的 attributes 仍可能被覆盖：

```text
Incoming Transaction
Address      = A
Cacheability = X
Shareability = Y
        ↓
       SMMU
   Bypass Address Translation
        ↓
Address      = A      ← 地址不变
Cacheability = X'     ← 可能被 Override
Shareability = Y'     ← 可能被 Override
```

所以：

> **Bypass Address Translation，不等于 Bypass Attribute Processing。**

具体允许覆盖哪些属性、输出如何编码，需要结合 Stream 配置、全局配置、Security State 和所用 SMMUv3 实现确认。

## 4. Stage 1 与 Stage 2 可以独立 Bypass

Stage 1 和 Stage 2 不是只能一起 Translate 或一起 Bypass，而是可以形成四种基本组合：

| Stage 1 | Stage 2 | 最终模式 |
|---|---|---|
| Bypass | Bypass | Full Bypass |
| Translate | Bypass | Stage 1 Only |
| Bypass | Translate | Stage 2 Only |
| Translate | Translate | Stage 1 + Stage 2 |

### Full Bypass

```text
Input Address
   ↓ S1 Bypass
   ↓ S2 Bypass
Output Address
```

### Stage 1 Only

```text
IOVA
  ↓ Stage 1
PA

Stage 2 = Bypass
```

### Stage 2 Only

```text
Device Address
      ↓ 直接作为 IPA
Stage 2
      ↓
PA
```

### Stage 1 + Stage 2

```text
IOVA
  ↓ Stage 1
IPA
  ↓ Stage 2
PA
```

因此，Bypass 是构造不同 Translation Mode 的基础，而不只是一个“完全关闭翻译”的开关。

## 5. 整个 SMMU Disabled 时的 Global Bypass

这与单个 Stream 的 Bypass 不同。

Per-Stream Bypass：

```text
SMMU Enabled
      ↓
根据 SID / STE 分别选择
Translate 或 Bypass
```

Global Bypass：

```text
SMMU Disabled
      ↓
所有 Stream
      ↓
全部使用 Bypass 行为
```

概念上：

```text
GPU  ─┐
DMA  ─┼→ SMMU Disabled → Global Bypass
PCIe ─┘
```

此时 Stream Table 和各 Stream 的 translation configuration 不再用于正常地址翻译。

## 6. SMMU Disabled 时的 GBPA

即使整个 SMMU 没有启用正常 translation，也不代表输出 attributes 完全不受控制。

培训材料指出，全局 Bypass 的 attribute override 可来自：

```text
SMMU_(S_)GBPA
```

可以先将 GBPA 理解成 Global Bypass Attribute configuration：

```text
SMMU Disabled
      ↓
Address 不翻译
      ↓
GBPA 决定适用的 Bypass Attribute 处理
      ↓
Output Transaction
```

因此：

> **即使 SMMU Disabled，Bypass transaction 的某些 attributes 仍可能受到全局 Bypass 配置控制。**

## 7. 为什么不同 Security State 要分别配置

Secure 与 Non-secure traffic 的安全要求不同，可以分别使用相应的 Bypass configuration：

```text
Secure Traffic
      ↓
Secure Bypass Configuration

Non-secure Traffic
      ↓
Non-secure Bypass Configuration
```

这与 SMMU 对 TrustZone 的支持一致：即使地址不执行 translation，也不能丢失 security state 带来的策略边界。

## 8. Per-Stream Bypass 与 Global Bypass 对比

| 对比项 | Per-Stream Bypass | Global Bypass |
|---|---|---|
| SMMU 状态 | Enabled | Disabled |
| 作用范围 | 由 SID/STE 选中的特定 Stream | 所有 Transaction |
| 其他 Stream | 仍可进行正常 Translation | 同样被迫 Bypass |
| 配置来源 | Stream/STE 相关配置 | Global Bypass 配置，例如 GBPA |
| 地址行为 | 不执行该 Stream 的正常 Translation | 不执行所有 Stream 的正常 Translation |
| 属性处理 | 仍可能 Override | 仍可能由全局配置 Override |

## 9. 实际使用示例

假设系统有两个 DMA：

```text
DMA0：地址已经是有效系统 PA，不需要 SMMU Translation
DMA1：使用 IOVA，需要地址隔离
```

可以配置：

```text
SMMU Enabled

SID 10 → Bypass
DMA0 PA
   ↓
直接作为下游地址访问 Memory

SID 20 → Stage 1 Translation
DMA1 IOVA
   ↓
SMMU Translation Table
   ↓
PA
```

这里要特别注意：使用 Bypass 意味着没有页表地址重映射和相应的 translation-based isolation，因此必须由系统集成和软件保证输入地址及安全策略是可信且正确的。

## 面试式回答

> **SMMU 的 Bypass 表示不执行地址翻译，输入地址直接作为下游地址使用，但 transaction 仍经过 SMMU，所以 cacheability、shareability 等 attributes 仍可能被 override。SMMU Enabled 时，Bypass 可以针对每个 Stream 独立配置；Stage 1 和 Stage 2 也能分别 Translate 或 Bypass，从而形成 Stage 1 only、Stage 2 only、两级 Translation 或 Full Bypass。若整个 SMMU Disabled，则所有 transaction 都进入 Global Bypass，相关 attributes 由 GBPA 等全局 Bypass 配置控制，并可针对不同 Security State 分别配置。**

## 一句话总结

> **Bypass = 不翻地址，不等于不经过 SMMU；地址可以不变，但 transaction attributes 和安全策略仍可能被处理。**

## 相关记录与资料

- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [Stage 2 Only Translation](17_Stage2_Only_Translation.md)
- [StreamID 与 Translation Context 映射](18_StreamID与Translation_Context映射.md)
- [SMMU 对 TrustZone 的支持](20_SMMU对TrustZone的支持.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

