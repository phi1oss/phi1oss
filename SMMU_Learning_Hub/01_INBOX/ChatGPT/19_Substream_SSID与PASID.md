# Substream、SSID 与 PCIe PASID

> 记录日期：2026-08-30  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 来源材料：培训截图，时间点未知  
> 状态：Inbox 问答整理，待结合 SMMUv3 与 PCIe 规范蒸馏

## 凝练问题

**SMMUv3 为什么需要 SubstreamID/SSID？同一 Stream 下为什么通常共享 Stage 2、却拥有独立的 Stage 1 Context 和 ASID？它与 PCIe PASID 有什么关系？**

## 回答

这部分的核心是：

> **SMMUv3 不仅能用 StreamID 区分不同设备或 traffic stream，还可以在同一个 Stream 内使用 SubstreamID（SSID）区分不同进程或地址空间。**

面试式一句话：

> **Substream 是一个 Stream 内进一步细分的 Translation Context。同一 Stream 的 Substreams 共用 Stream 级的 Stage 2 configuration，但每个 Substream 可以拥有独立的 Stage 1 Context、页表和 ASID。**

## 1. 为什么需要 Substream

如果没有 Substream，通常是：

```text
一个 StreamID
    ↓
一套 Translation Context
    ↓
一套 Stage 1 Page Table
```

这对于普通 DMA engine 可能足够，但 GPU、NPU、PCIe accelerator 等设备可能同时替多个进程工作：

```text
GPU
├─ Process A
├─ Process B
└─ Process C
```

它们来自同一个 GPU，因此 StreamID 相同；但每个进程需要独立的虚拟地址空间：

```text
Process A: IOVA 0x1000 → IPA_A
Process B: IOVA 0x1000 → IPA_B
```

只靠 StreamID 无法区分，于是需要：

```text
StreamID + SubstreamID
```

例如：

```text
GPU StreamID = 20

SSID = 0 → Process A
SSID = 1 → Process B
SSID = 2 → Process C
```

## 2. 为什么共享 Stage 2、独立 Stage 1

假设 GPU 被分配给 VM1，其中两个 Application 使用不同 Substream：

```text
                 GPU
            StreamID = 20
                  │
        ┌─────────┴─────────┐
        │                   │
      SSID=0              SSID=1
    Application A       Application B
```

两个 Application 属于同一个 VM，因此最终都受同一个 VM 的 Guest IPA→Host PA 隔离控制：

```text
                 Shared Stage 2
                        │
              ┌─────────┴─────────┐
              │                   │
         Substream 0          Substream 1
              │                   │
         Stage 1 A           Stage 1 B
              │                   │
      Application A PT      Application B PT
```

因此：

- Stage 1 区分同一个 VM 内的不同进程或地址空间。
- Stage 2 限制该 VM 最终能够访问哪些 Host physical memory。

可以记成：

> **Stage 1 分进程，Stage 2 隔离 VM。**

## 3. 一笔带 SSID 的 Transaction 如何转译

假设 GPU 发出：

```text
StreamID    = 20
SubstreamID = 1
Address     = 0x4000
```

概念流程是：

```text
SID = 20
   ↓
选择 GPU 对应的 STE
   ↓
获得该 Stream 的公共 Stage 2 配置
   ↓
SSID = 1
   ↓
从 CD Table 选择 Substream 1 的 CD
   ↓
获得 Stage 1 Page Table B
   ↓
IOVA 0x4000
   ↓ Stage 1 B
IPA
   ↓ 公共 Stage 2
PA
```

如果改为 `SSID=0`，SMMU 会选择另一个 CD 和 Stage 1 Page Table。即使输入 IOVA 相同，也可以产生完全不同的 Stage 1 translation。

## 4. 放入 STE/CD 层级理解

```text
                StreamID
                    ↓
                   STE
          ┌─────────┴─────────┐
          │                   │
 Stage 2 Configuration    CD Table Pointer
                                  │
                             SubstreamID
                                  ↓
                                 CD
                                  ↓
                        Stage 1 Configuration
                                  ↓
                         Stage 1 Page Table
```

对应关系是：

```text
SID  → 选择 STE
SSID → 从 CD Table 选择 CD
STE  → Stream 级总体配置，包含 Stage 2 与 CD 定位信息
CD   → Substream 的 Stage 1 配置
```

## 5. 每个 Substream 为什么通常有自己的 ASID

原因与 CPU 进程使用 ASID 类似：

```text
SSID 0 → Process A → ASID 5
SSID 1 → Process B → ASID 8
```

SMMU/TLB 可以区分：

```text
ASID=5, IOVA=0x4000 → IPA_A
ASID=8, IOVA=0x4000 → IPA_B
```

这样不同进程使用相同虚拟地址时，缓存的 translation 不会混淆。SSID 用来选择 CD，ASID 则参与标识该 Stage 1 translation context 及其缓存结果；两者职责相关但不相同。

## 6. 与 PCIe PASID 的关系

PASID 是 Process Address Space ID。支持 PASID 的 PCIe Device 可以在 transaction 中携带进程级地址空间身份：

```text
PCIe Device
   ├─ PASID 10 → Process A
   └─ PASID 20 → Process B
```

进入 Arm SMMU 系统后，这种进程级身份可以被系统集成为 SMMU 的 SubstreamID/SSID 选择信息：

```text
PCIe PASID
   ↓ 系统集成映射
SSID / Substream Selection
   ↓
对应的 Stage 1 Context
```

因此：

> **PASID 是 PCIe 世界的进程地址空间标签，SSID 是 SMMU 用来选择 Substream Translation Context 的标识。**

但不能假设两者在所有系统中 bit-by-bit 永远相同，具体映射属于 PCIe Root Complex、互连和 SMMU 的系统集成。

## 面试式回答

> **Substream 是 SMMUv3 在一个 Stream 内进一步区分多个 Translation Context 的机制。StreamID 通常标识设备或 traffic stream，SubstreamID 则区分同一设备中的不同进程或地址空间。同一 Stream 下的 Substreams 使用共同的 Stream/Stage 2 configuration，但可以分别拥有独立的 Stage 1 page table 和 ASID。在 SMMUv3 中，StreamID 先选择 STE，SSID 再选择 Context Descriptor，由 CD 定义该 Substream 的 Stage 1 translation。PCIe PASID 是典型的进程级身份来源，但 PASID 到 SSID 的具体映射由系统集成决定。**

## 一句话总结

> **SID = 哪个设备；SSID = 设备里的哪个进程；STE = Stream 总体配置；CD = Substream 的 Stage 1 配置。**

## 相关记录与资料

- [StreamID 与 Translation Context 映射](18_StreamID与Translation_Context映射.md)
- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [Stage 2 Only Translation](17_Stage2_Only_Translation.md)
- [STE 与 CD 的关系](03_STE与CD的关系.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

