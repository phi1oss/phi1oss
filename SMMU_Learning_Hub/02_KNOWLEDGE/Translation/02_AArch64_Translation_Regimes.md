# AArch64 Translation Regimes

> 整理日期：2026-08-30  
> 状态：正式知识文档，待脱稿解释验证

## 核心结论

**Translation regime 是某个执行上下文的一整套地址翻译环境，不是一张具体 page table。它由 translation tables 以及 TTBR、TCR、MAIR、SCTLR 等寄存器共同定义，并规定输入地址范围、granule、walk 方式、memory attributes、权限和是否涉及 Stage 2。**

最重要的区分是：

> **Translation regime 是“翻译环境”；Stage 1/Stage 2 是“翻译步骤”。**

## 1. 一个 Regime 包含什么？

```text
Translation Regime
├─ TTBR   → translation table base、ASID 等 context
├─ TCR    → VA size、granule、walk cacheability/shareability 等
├─ MAIR   → AttrIndx 对应的 Normal/Device memory encoding
├─ SCTLR  → MMU enable 和相关控制
└─ Translation tables
```

所以“换了一张页表”和“定义了一个 translation regime”不是同一件事。进程切换可能只是在同一 architecture-defined regime 中切换 active TTBR 与 ASID。

## 2. 为什么需要多个 Regimes？

不同 Exception Level 和 Security state 需要彼此隔离的地址空间与控制权：

```text
EL0/EL1 software
→ EL1&0 translation regime，由 EL1 管理

EL2 Hypervisor
→ EL2 translation regime

EL3 Secure Monitor
→ EL3 translation regime
```

EL0 与 EL1 通常不是两套完全独立的 Stage 1 regime。它们共享由 EL1 配置的 translation environment，再通过 TTBR0/TTBR1、AP、UXN/PXN 和 PAN 等机制区分用户与内核空间和权限。

## 3. TTBR0 与 TTBR1

EL1&0 Stage 1 通常使用：

```text
TTBR0_EL1 → 常用于用户/应用低地址区域
TTBR1_EL1 → 常用于内核高地址区域
```

两个区域大小由 `TCR_EL1.T0SZ/T1SZ` 控制。VA 中间未被任一区域覆盖的部分会 fault。具体选择边界取决于 TCR 配置，不应只背固定地址常量。

## 4. Stage 1 与 Stage 2

在普通非虚拟化场景中：

```text
VA → Stage 1 → PA
```

在 Guest 场景中：

```text
Guest VA
   ↓ Stage 1，Guest OS 管理
IPA
   ↓ Stage 2，Hypervisor 管理
PA
```

Guest OS 可以控制 `VA → IPA`，但不能直接决定真正的系统 PA。Stage 2 让 Hypervisor 实现 VM 隔离、内存重映射和资源控制。

Hypervisor 自身在 EL2 执行时有自己的 EL2 Stage 1 translation：

```text
Hypervisor VA → EL2 Stage 1 → PA
```

不能简单认为所有 EL2 access 都自动经过 Guest 的 Stage 2。

## 5. ASID、Global 与 Context Switch

Non-global TLB entry 通常需要匹配当前 ASID；Global entry 可跨 ASID 使用，常用于共享内核 mapping。

```text
Process A: TTBR_A + ASID 1
Process B: TTBR_B + ASID 2
```

ASID 允许两个进程的 translation 同时保留在 TLB，减少 context switch 时全量 invalidation 的需求。

## 6. 与 SMMU Translation Context 的边界

CPU 的 translation regime 由 EL、Security state 和系统寄存器体系定义。SMMU 也执行 Stage 1/Stage 2 translation，但通过 StreamID、STE、Context Descriptor 等设备请求上下文选择配置。

二者共享很多页表和属性概念，但不能把 CPU 的 EL1 TTBR 选择机制直接等同于 SMMU 的 Stream Table lookup。

## 面试表达

> **Translation regime 是 Arm 为某个执行上下文定义的一整套地址翻译环境，包括使用哪套页表、VA size、translation granule、memory attributes 和相关权限，主要由 TTBR、TCR、MAIR、SCTLR 等配置。不同 EL 和 Security state 可以有独立 regime；虚拟化时 Guest VA 先经 Stage 1 得到 IPA，再由 EL2 管理的 Stage 2 得到 PA。**

## 来源与相关资料

- [MMU.pdf](../../99_SOURCE/Training/MMU.pdf)：PDF 第 7–9、15–18、23 页，translation regimes、TTBR0/1、ASID、权限与配置。
- [Inbox 14：Translation Regime 面试问答](../../01_INBOX/ChatGPT/14_Translation_Regime面试问答.md)。
