# 多进程如何使用 TTBR0_EL1

> 记录日期：2026-09-01  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 状态：Inbox 问答整理，待并入 CPU Stage-1 Translation Context 主线

## 凝练问题

**CPU 执行 Table Walk 时，多个用户态进程是否共用同一个 `TTBR0_EL1` 来寻找页表起始地址？不同进程的用户页表如何被选择？**

## 核心结论

> **不同用户进程通常各自拥有独立的用户态 Stage-1 页表。每个 PE/Core 只有一个当前生效的 `TTBR0_EL1`，它保存当前 Active 用户地址空间的页表根和 ASID；发生进程 Context Switch 时，OS 通常把新进程对应的页表根与 ASID装入该 PE 的 `TTBR0_EL1`。**

因此，不是所有进程固定共用一个页表基地址，而是：

```text
Memory 中同时存在：

Process A Page Table
Process B Page Table
Process C Page Table
...

当前 PE 的 TTBR0_EL1
        ↓
只选择当前正在该 PE 上运行的进程页表
```

## 1. TTBR0_EL1 是当前用户地址空间的入口

例如两个进程使用相同虚拟地址，但映射到不同物理地址：

```text
Process A:
VA 0x4000 → PA 0x8000
PageTable_A Base = 0x1000_0000
ASID = 1

Process B:
VA 0x4000 → PA 0xA000
PageTable_B Base = 0x2000_0000
ASID = 2
```

Core0 运行 Process A 时：

```text
TTBR0_EL1
├─ Base = 0x1000_0000
└─ ASID = 1

VA 0x4000
   ↓
PageTable_A
   ↓
PA 0x8000
```

调度器把 Core0 切换到 Process B 后，OS 将 Translation Context 切换为：

```text
TTBR0_EL1
├─ Base = 0x2000_0000
└─ ASID = 2
```

此时同一个 `VA 0x4000` 的 Table Walk 变为：

```text
VA 0x4000
   ↓
TTBR0_EL1
   ↓
PageTable_B
   ↓
PA 0xA000
```

所以 `TTBR0_EL1` 更准确的含义是：

> **当前这个 PE 正在使用的用户地址空间入口。**

## 2. 未运行进程的页表仍保存在内存中

没有被当前 PE 执行的进程，其页表不会消失。OS 会在进程相关的软件数据结构中保存页表根、ASID 等 Translation Context，待该进程再次被调度时恢复。

硬件关系不是：

```text
TTBR0_EL1
 ├→ Process A Page Table
 ├→ Process B Page Table
 └→ Process C Page Table
```

而是：

```text
一个 PE 上的 TTBR0_EL1
          ↓
当前只选择一个 Active Address Space
```

## 3. ASID 为什么重要

如果每次切换 `TTBR0_EL1` 都清空整个 TLB，开销会很大。ASID 用于区分不同地址空间的 Cached Translation，使相同 VA 可以同时保留不同进程的映射：

```text
TLB

ASID=1, VA=0x4000 → PA=0x8000
ASID=2, VA=0x4000 → PA=0xA000
```

当前 ASID 为 2 时，不会误命中 ASID 为 1 的 Translation。因此可以粗略理解为：

> **`TTBR0_EL1.Base` 决定“从哪里开始 Table Walk”，ASID 决定“这条 Translation 属于哪个地址空间”。**

这使 OS 在进程切换后不一定需要清除其他进程的全部 TLB Entry；具体 TLB 维护仍取决于 ASID 分配、复用和页表修改情况。

## 4. 多核系统中的 TTBR0_EL1

`TTBR0_EL1` 是每个 PE 的架构状态，不是整个 SoC 只有一个实例：

```text
Core0                         Core1
运行 Process A                运行 Process B

TTBR0_EL1                     TTBR0_EL1
     ↓                             ↓
PageTable_A                   PageTable_B
```

因此不同用户进程可以同时运行在不同 Core 上，各自使用不同的页表和地址空间。

## 5. TTBR0_EL1 与 TTBR1_EL1 的典型分工

典型 OS 常采用：

```text
TTBR0_EL1 → 当前用户进程的低地址空间
TTBR1_EL1 → Kernel 高地址空间
```

于是进程切换时通常主要切换 `TTBR0_EL1` 所选择的用户页表，而 Kernel Mapping 可以保持共享或高度共享。

这是常见的软件组织方式，不应表述为所有系统都必须采用的架构强制规则。

## 6. 与 SMMU Translation Context 的对比

CPU 的选择方式：

```text
Scheduler 选择当前 Process
          ↓
软件切换 TTBR0_EL1 + ASID
          ↓
选择当前 PE 的用户 Translation Context
```

SMMU 的选择方式：

```text
Device Transaction 携带 SID / SSID
               ↓
Stream Table / STE / CD
               ↓
选择相应的 Device/Process Translation Context
```

根本差异是：

- 一个 PE 同一时刻只执行当前进程对应的 Translation Context，因此可以由 Scheduler 在 Context Switch 时切换 `TTBR0_EL1`。
- SMMU 可能并发接收多个设备和进程的 Transaction，因此需要用 SID/SSID 动态选择 STE/CD，而不是依赖单个当前上下文寄存器。

## 面试式回答

> **多个用户进程通常不会共用同一个固定的 `TTBR0_EL1` 页表基地址。每个进程拥有自己的用户态 Stage-1 Page Table；每个 PE 有一个当前生效的 `TTBR0_EL1`，OS 在进程 Context Switch 时将新进程的页表根地址和 ASID 装入该寄存器。其他进程的页表仍保存在内存中，但没有被当前 PE 选中。多核系统中每个 PE 又有自己的 `TTBR0_EL1`，所以不同 Core 可以同时运行不同进程和地址空间。**

## 一句话总结

> **每个进程有自己的用户页表；每个 Core 只有一个当前生效的 `TTBR0_EL1`，Context Switch 时由 OS 将它切换到新进程的页表根，并用 ASID 区分 TLB 中不同地址空间的 Translation。**

## 相关记录与知识

- [Substream、SSID 与 PCIe PASID](19_Substream_SSID与PASID.md)
- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [AArch64 Translation Regimes](../../02_KNOWLEDGE/Translation/02_AArch64_Translation_Regimes.md)
- [Stage 1 配置与 TLB 维护](../../02_KNOWLEDGE/Software/01_Stage1配置与TLB维护.md)

