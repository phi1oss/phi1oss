# Stage 2 Only Translation

> 记录日期：2026-08-30  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 状态：Inbox 原始问答，待结合 SMMUv3 规范进一步蒸馏

## 凝练问题

**如何理解 SMMU 的 Stage 2 only translation？它为什么不需要 Stage 1，适用于什么场景，又提供了什么保护？**

## 回答

可以把 **Stage 2 only translation** 理解成：

> **设备发出的地址已经是 Guest 视角的“物理地址” IPA，因此 SMMU 不再做 Stage 1 的 IOVA→IPA，只做 Stage 2 的 IPA→真实 PA。**

在 MMU-720AE/SMMUv3 中，STE 可以明确配置成：

```text
Stage 1 bypass
Stage 2 translation
```

即 `STE.Config = 0b110`。

## 1. 与两级 Translation 对比

完整两级 translation：

```text
Device IOVA
    ↓
Stage 1
    ↓
IPA
    ↓
Stage 2
    ↓
PA
```

Stage 2 only：

```text
Device DMA Address
       │
       │ 直接被当成 IPA
       ▼
    Stage 2
       ↓
      PA
```

所以它省掉的是：

> **设备自身那一级 IOVA 地址虚拟化。**

## 2. 为什么会有这种模式

典型场景是：

> **Device 被分配给 Guest，Guest 直接使用自己的 Guest Physical Address 编程 DMA。**

例如 Guest OS 认为自己拥有：

```text
Guest RAM:
0x8000_0000 ~ 0x8FFF_FFFF
```

Guest driver 给设备设置：

```text
DMA address = 0x8000_1000
```

Guest 认为这是自己的物理地址，但在 Host 系统中它只是：

```text
IPA = 0x8000_1000
```

Hypervisor 可以通过 Stage 2 将其映射为：

```text
PA = 0x4800_01000
```

完整路径是：

```text
Device
DMA = 0x8000_1000
        ↓
      SMMU
        ↓
Stage 2 Page Table
        ↓
PA = 0x4800_01000
        ↓
       DDR
```

## 3. Stage 2 Only 解决的核心问题

它主要不是解决 Device 的 IOVA virtualization，而是解决：

> **Guest Device 对真实物理内存的隔离和重映射。**

例如两个 Guest 都可以认为 `IPA 0x80000000` 属于自己，但 Hypervisor 可以用不同的 Stage 2 Context 将它们映射到不同的 Host PA：

```text
Guest 1:
IPA 0x80000000
    ↓
PA 0x400000000

Guest 2:
IPA 0x80000000
    ↓
PA 0x500000000
```

因此相同的 Guest Physical Address 不会让两个 VM 访问同一块真实 DDR。

## 4. 为什么不需要 Stage 1

Stage 1 本来主要解决：

```text
Device IOVA
      ↓
设备自己的虚拟地址空间
      ↓
IPA
```

如果 Guest driver 本身就直接用 IPA 编程设备：

```text
Device Address = Guest Physical Address
```

那么就没有 IOVA→IPA 这一级需要转换，因此：

```text
Stage 1 bypass
Stage 2 enable
```

已经足够。

## 5. 与 CPU 虚拟化放在一起理解

CPU Guest：

```text
Application VA
      ↓
CPU Stage 1
      ↓
Guest IPA
      ↓
CPU Stage 2
      ↓
Host PA
```

Stage-2-only Device：

```text
Guest Driver
直接给 Device 一个 IPA
      ↓
Device DMA
      ↓
SMMU Stage 2
      ↓
Host PA
```

所以可以理解成：

> **CPU 还需要把程序 VA 翻成 IPA；而该 Device 已经直接使用 IPA 发出 DMA，因此 SMMU 只需要完成后半段。**

## 6. Stage 2 Only 仍然提供访问保护

没有 Stage 1 不代表没有 SMMU 保护：

```text
Device DMA
    ↓
IPA
    ↓
Stage 2 Permission Check
    ↓
允许？
 ├─ Yes → PA
 └─ No  → Fault
```

Hypervisor 仍然可以限制：

> **这个 Guest Device 能够 DMA 哪些 Host physical pages。**

这正是 Device passthrough 和虚拟化场景中的重要隔离边界。

## 面试式回答

> **Stage 2 only 表示 SMMU bypass Stage 1，只执行 Stage 2 translation。此时设备发出的 DMA 地址直接被视为 IPA，也就是 Guest Physical Address，再由 Hypervisor 管理的 Stage 2 page table 转换成真实 PA。典型场景是设备分配给 Guest，而 Guest driver 直接使用自己的物理地址编程设备。虽然没有 Stage 1 的 IOVA→IPA 虚拟化，但 Stage 2 仍然负责 Guest Device 到 Host physical memory 的重映射和访问隔离。**

## 一句话总结

> **Stage 1 only：设备地址虚拟化；Stage 2 only：Guest Device 的物理地址隔离；Stage 1+2：两者都要。**

## 相关记录与资料

- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [STE 与 CD 的关系](03_STE与CD的关系.md)
- [Translation Regime 面试问答](14_Translation_Regime面试问答.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

