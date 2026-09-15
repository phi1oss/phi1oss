# Arm RME 扩展概述

## 问题

什么是 Arm RME？它解决什么问题？Root、Realm、RMM、GPT/GPC、SMMU 和 RME Device Assignment 之间有什么关系？

## 核心结论

**RME（Realm Management Extension）是 Arm 为机密计算（Confidential Computing）引入的一套体系结构扩展。它在原来的 Secure/Non-secure 之外增加 Realm 和 Root 安全状态，并通过 RMM、GPT/GPC、SMMU 等机制保护 Realm 的内存和设备访问。**

传统 TrustZone 与 RME 的安全状态可以先这样对比：

```text
传统 Arm TrustZone
-----------------
Secure
Non-secure

加入 RME 后
-----------------
Root
Realm
Secure
Non-secure
```

其中最关键的是 Realm：

> **Realm 是受硬件保护的隔离执行环境。即使 Host OS、Hypervisor 或部分高权限软件被攻破，也不能随意读取 Realm 内部的数据。**

## 1. 为什么需要 RME？

传统虚拟化中的权限关系通常是：

```text
Guest VM
   ↓
Hypervisor
   ↓
Hardware
```

Hypervisor 拥有很高权限，理论上可以：

```text
读取 Guest Memory
修改 Guest State
检查 Guest Page Table
```

如果用户希望实现：

> **“即使云服务商的 Hypervisor 也不能看到我的 VM 数据。”**

传统虚拟化隔离就不够了。

RME 为此增加 Realm 这一硬件隔离域：

```text
Host OS / Hypervisor
         │
         × 不能直接访问
         │
         ▼
       Realm
```

因此，Realm 不是简单的另一个普通 VM，而是用于承载 confidential workload 的硬件保护环境。

## 2. Realm、Root 和 RMM 分别是什么？

### Realm

Realm 是运行机密工作负载的隔离执行域，其代码和数据受到硬件保护，普通 Host/Hypervisor 不能直接访问。

### Root

Root 是 RME 体系中负责管理全局安全边界的高权限安全状态。它可以管理 Realm 依赖的关键安全资源，例如 GPT。

概念上可以理解为：

```text
Root
 ├─ 管理 GPT
 ├─ 管理 Realm 的创建和销毁
 └─ 管理 Realm 使用的部分安全资源
```

一般业务软件不会直接运行在 Root。

### RMM

```text
RMM = Realm Management Monitor
```

RMM 是 RME 管理体系中的重要软件组件，负责 Realm 的生命周期和资源管理。

三者可以简化为：

```text
Root
  │ 管理 RME 安全边界
  ▼
RMM
  │ 管理 Realm 生命周期与资源
  ▼
Realm
  │ 运行 Confidential Workload
```

## 3. GPT 与 GPC：保护物理内存归属

普通 translation table 回答：

```text
VA / IOVA
    ↓
   PA
```

RME 还需要额外回答：

> **“这个物理地址属于哪个安全世界？”**

### GPT

```text
GPT = Granule Protection Table
```

GPT 按 physical granule 记录其所属的 Physical Address Space（PAS）：

```text
Physical Address Range
          ↓
         GPT
          ↓
Root / Realm / Secure / Non-secure PAS
```

因此：

```text
Translation Table
→ 决定地址在哪里

GPT
→ 决定这块物理内存在安全意义上属于谁
```

### GPC

```text
GPC = Granule Protection Check
```

GPC 根据访问者属性和 GPT 中记录的 PAS 归属，判断访问是否合法。

例如：

```text
Non-secure Requester
          │
          │ 访问 PA = 0x9000_1000
          ▼
         GPC
          │
          │ GPT：目标 Granule 属于 Realm PAS
          ▼
         DENY
```

因此，即使某笔 transaction 不需要地址翻译，也仍可能需要经过 GPC，以保证它不能越权进入其他 PAS。

## 4. RME 与 SMMU 的关系

CPU 对 Realm memory 的访问可以由 CPU/MMU 和 RME memory system 保护，但 Device DMA 可能绕开 CPU，直接访问物理内存：

```text
Device
   │ DMA
   ▼
Physical Memory
```

因此，RME 需要在设备侧设置安全检查点。SMMU 在这里承担：

> **Device-side Memory Protection Enforcement Point**

其基本流程是：

```text
Device
   ↓
  SMMU
   │
   ├─ Stage 1 / Stage 2 Translation（需要时）
   │
   └─ GPC
        ↓
       GPT
        ↓
检查目标 PA 属于哪个 PAS
        ↓
   Allow / Fault
```

所以在 RME 系统中，SMMU 不再只是 DMA 地址翻译器，而是：

> **设备侧的 Translation + Security Isolation Gateway。**

这也是 MMU-720AE TRM 中出现以下能力或字段的背景：

```text
ROOT_IMPL
REALM_IMPL
RME_IMPL
RME_DA_IMPL
BGPTM
MECID
GPC
GPT Cache
```

## 5. RME Device Assignment

Realm workload 也可能需要使用 PCIe Device、Accelerator 或 NIC。RME 不能只保护 CPU 和内存，还必须支持设备被安全地分配给 Realm。

```text
RME-DA
= RME Device Assignment
```

其核心过程可以理解为：

```text
Physical Device / Device Interface
              ↓
安全地分配给某个 Realm
              ↓
SMMU 建立相应的 Translation / Protection Context
              ↓
Device 只能访问授权给该 Realm 的资源
```

因此，“Realm Device”并不表示某个硬件天生属于 Realm。更准确的含义是：

> **某个 Device Interface 被安全地 assignment 给 Realm，并由 SMMU 等硬件强制执行其访问边界。**

这与 MMU-720AE 中的：

```text
RME_DA_IMPL = 1
```

直接相关。

## 整体关系

```text
                    Arm RME
                       │
          ┌────────────┼────────────┐
          │            │            │
        Root         Realm         RMM
          │            │
          │       Confidential
          │         Workload
          │
          ▼
         GPT
          │
  记录每个 Physical Granule
        所属的 PAS
          │
          ▼
         GPC
      检查访问是否合法
          │
     ┌────┴────┐
     ▼         ▼
  CPU/MMU     SMMU
                 │
               Device
```

## 一句话总结

> **RME 为 Arm 增加 Realm 机密执行环境；Root/RMM 管理安全边界和 Realm 生命周期，GPT 记录物理内存归属，GPC 强制检查访问，SMMU 则把这种保护扩展到 DMA 和设备侧。**
