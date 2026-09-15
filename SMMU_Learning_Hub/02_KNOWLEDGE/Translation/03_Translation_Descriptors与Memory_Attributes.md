# Translation Descriptors 与 Memory Attributes

> 初次整理日期：2026-08-30  
> 更新日期：2026-09-10  
> 状态：正式知识文档，待属性配置练习验证

## 核心结论

**Leaf descriptor 不只给出 output address，还定义访问权限、执行权限、memory type、cacheability、shareability、安全属性和访问状态。Table walk 的最终目标，是把 descriptor 与 MAIR、TCR 等配置解析成完整 translation result，再缓存到 TLB。**

## 1. Descriptor 类型

AArch64 translation table entry 为 64 bit，低位编码决定其基本类型：

- **Table descriptor**：指向下一级表。
- **Block descriptor**：在允许的中间 level 形成较大 mapping。
- **Page descriptor**：在最后一级形成基础 granule 大小 mapping。
- **Fault/Invalid descriptor**：不存在有效 mapping。

Block/Page descriptor 可概念化为：

```text
Upper Attributes | Output Address | Lower Attributes | Type
```

## 2. 主要 Upper Attributes

### UXN/PXN

- UXN：Unprivileged Execute Never，限制 EL0 执行。
- PXN：Privileged Execute Never，限制特权执行。

Device/MMIO 区域通常应配置为 Execute Never，避免 speculative instruction fetch。

### Contiguous Bit

Contiguous Bit 声明一组相邻 descriptor 的 VA、PA 连续且属性一致，允许硬件扩大有效 TLB coverage。

以 4KB granule 为例：

```text
16 × 4KB adjacent pages
        ↓
64KB contiguous translation group
```

页表中仍然是 16 个独立 PTE；架构也不强制 TLB 必须物理合并成一条 SRAM entry。它规定使用条件和可观察行为，不规定具体微架构。

### DBM

Dirty Bit Modifier 可与访问权限配合，让硬件在支持和启用相应能力时更新页面 dirty/write 状态，减少软件 fault 处理。

## 3. 主要 Lower Attributes

### AttrIndx 与 MAIR

Descriptor 不直接存放完整 memory type，而是用 `AttrIndx` 选择 `MAIR_ELx` 中的一个 8-bit entry：

```text
Descriptor.AttrIndx
        ↓
MAIR_ELx[n]
        ↓
Normal / Device + Cache Policy
```

### AP

AP 控制 privileged/unprivileged 的读写权限。权限检查发生在 translation result 已找到之后，因此 mapping 存在仍可能产生 Permission Fault。

### AF

AF 表示 block/page 是否已经被访问。`AF=0` 且硬件自动更新未启用时，访问产生 Access Flag Fault；若 HA 等配置和实现允许，硬件可原子更新 AF。

### nG 与 ASID

- `nG=0`：Global translation，可跨 ASID 使用。
- `nG=1`：Non-global translation，TLB lookup 需要匹配当前 ASID。

### SH

Shareability 描述 memory location 对哪些 observer 共享，例如 Non-shareable、Inner Shareable、Outer Shareable。它需要与系统实际 coherency domain 匹配。

### NS 与安全地址空间

Training 模块使用 Secure/Non-secure 的基础模型解释 NS 属性。支持 RME 的完整 Armv9 系统还会涉及 Root/Realm 等 PAS，不能把基础 slide 的二分模型扩展成所有实现的完整结论。

## 4. Normal 与 Device Memory

### Normal Memory

适用于代码、栈、堆、普通数据和 DMA buffer 数据区。允许 cache、speculative read、访问合并和一定程度的乱序；具体行为由 memory attributes 和 ordering rules 限制。

### Device Memory

适用于 MMIO 和有副作用的 peripheral register。它通过 G/R/E 属性限制访问行为：

- Gathering：是否允许多个访问合并。
- Reordering：是否允许对同一设备区域重排。
- Early Write Acknowledgement：写响应是否允许由中间缓冲提前返回。

最严格的 `Device-nGnRnE` 常用于控制寄存器。

```text
Executable code → Normal
Stack / Heap    → Normal
DMA data buffer → Normal，控制寄存器仍是 Device
GIC/UART/MMIO   → Device
```

## 5. TLB 缓存的是解析结果

一次 walk 结束后，TLB entry 概念上缓存：

```text
VA Tag + ASID/Global
Output PA Base
Mapping/Coverage Size
AP + UXN/PXN
Resolved Memory Type
Shareability
Security Context
Valid/State information
```

因此后续 TLB hit 不需要重新读取 PTE 或 MAIR。不同 entry 可以覆盖 4KB Page、64KB contiguous group、2MB Block 或 1GB Block；lookup 根据 coverage size 使用不同 tag/offset 边界。

## 6. Contiguous Bit 使用条件

一组 descriptor 必须满足：

- VA 连续、PA 连续。
- 处于相同 translation level。
- 相关 attributes 一致。
- group 起始 VA/PA 满足对齐要求。
- group 数量符合 granule 和 level 的架构规定。
- 整组一致设置 Contiguous Bit。

违反这些条件属于 programming error，可能造成 abort 或属性不一致。

## 易混淆点

- Translation granule 是基础页表粒度；mapping/TLB coverage 可以更大。
- Block descriptor 是一个大 mapping；Contiguous group 仍由多个小 descriptor 表示。
- Cacheability 与 Shareability 不是同一属性，但必须与系统 coherency 设计协调。
- AttrIndx 只是索引；真正的 memory type encoding 位于 MAIR。


## 2026-09-10 补充：Shareability、Cacheability 与 AttrIndx

依据 MMU.pdf 第 16–18 页及最近问答：

| 概念 | 回答的问题 | 主要载体（本培训 Stage 1 模型） |
|---|---|---|
| Shareability | 哪个 observer 范围需要相应共享/一致性语义？ | PTE.SH |
| Cacheability | Inner/Outer cache 行为如何？ | MAIR 的所选属性项 |
| AttrIndx | 使用哪一项属性？ | PTE 中的索引 |

Shareability 的 Inner/Outer 描述 domain，Cacheability 的 Inner/Outer 描述缓存行为层次；二者不能相等，也不固定对应“同 cluster/跨 cluster”或“L1/L3”。培训第 17 页的两幅拓扑中，L2 可处于不同侧，正是为了说明行为抽象而非固定物理编号。

例如 AttrIndx=3 只表示选择 MAIR 第 3 项，不直接意味着 WB 或 NC；必须读取该项内容。PTE.SH 并不是从这个 MAIR 项中读出来的。

上述区分不意味着所有属性组合在所有 memory type、regime 与接口上都合法。Non-shareable 也不是访问权限，不能保证其他 observer 不访问该位置。总线侧 AxCACHE/AxDOMAIN 及具体归一化见 [属性转换边界](../Interfaces/08_AxCACHE_AxDOMAIN与属性转换边界.md)。

CD.IR/OR/SH 控制页表遍历自身的访问，PTE.AttrIndx/SH 控制映射数据的属性，作用对象不同。

补充来源：[Inbox 60](../../01_INBOX/ChatGPT/60_Shareability_Cacheability与AttrIndx.md)、[Inbox 62](../../01_INBOX/ChatGPT/62_AxCACHE与AxDOMAIN的含义.md)。

自测：Inner Shareable 是否等于 Inner Cacheable？为什么 AttrIndx 不是重复存储缓存策略？

## 一句话总结

> **Descriptor 决定“映射到哪里以及能怎样访问”，MAIR 等寄存器补全 memory type；TLB 缓存的是这些信息解析后的完整 translation result。**

## 来源与相关资料

- [MMU.pdf](../../99_SOURCE/Training/MMU.pdf)：PDF 第 10–23 页，descriptor、权限、AF、ASID、shareability、cacheability 与 memory types。
- [Inbox 11：Contiguous Bit 与 TLB Coverage](../../01_INBOX/ChatGPT/11_Contiguous_Bit与TLB覆盖范围.md)。
- [Inbox 12：Contiguous Bit 下的 TLB 命中与转换](../../01_INBOX/ChatGPT/12_Contiguous_Bit下的TLB命中与地址转换.md)。
