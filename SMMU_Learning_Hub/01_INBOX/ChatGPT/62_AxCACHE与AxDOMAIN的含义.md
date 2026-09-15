# AxCACHE 与 AxDOMAIN 的含义

记录日期：2026-09-10
记录端：本地

来源：[SMMU](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)。以下保留原回答，去除不可解析的原引用标记；查证后的边界以关联正式知识为准。

## 问题

如何理解 axi 中的 axdomain 和 axcache 信号

问答日期：2026-09-10；原问答标识：`bbb21592-2bba-4331-abdf-be29e007fd6e`。

## 回答

可以把这两个信号先抓成一句话：

> **`AxCACHE` 决定“这笔访问的数据该怎么缓存”；`AxDOMAIN` 决定“这笔访问在哪个范围内参与一致性”。**

### 1. `AxCACHE`：描述 memory/cache 行为

`AxCACHE` 是 4-bit 属性，核心描述这笔 transaction 的：

- 是否可 Buffer
- 是否允许修改/合并等
- 是否具备 Read Allocate / Write Allocate 属性

组合起来可以表达类似：

```text
Device / Strongly-ordered 类访问
Normal Non-cacheable
Normal Write-Through
Normal Write-Back
...
```

所以它回答的是：

> **“这笔数据进入 memory system 后，cache 可以怎么处理它？”**

例如普通 DDR 数据可能是：

```text
Normal + Write-Back Cacheable
```

而 MMIO 寄存器通常是：

```text
Device / Non-cacheable
```

---

### 2. `AxDOMAIN`：描述 coherency scope

`AxDOMAIN` 是 ACE/ACE-Lite 的一致性属性，用来说明：

> **这笔 transaction 需要在哪个 shareability domain 内保持一致。**

概念上：

```text
Non-shareable
Inner Shareable
Outer Shareable
System
```

例如：

```text
CPU Cluster 内多个 Core
        ↓
Inner Shareable Domain

多个 Cluster / coherent agent
        ↓
Outer Shareable Domain
```

所以它回答：

> **“哪些 coherent observer 需要参与这笔访问的一致性处理？”**

---

### 3. 两者最容易混的地方

假设：

```text
AxCACHE → Cacheable
AxDOMAIN → Inner Shareable
```

它表达的是两件事：

```text
AxCACHE:
“这份数据允许缓存”

AxDOMAIN:
“而且要在 Inner Shareable 范围内保持一致”
```

不能理解成：

```text
Inner Shareable = Inner Cacheable
```

这两个完全不是一个维度。

---

### 4. 一个具体例子

假设 CPU 和 DMA 都会访问一块共享 buffer：

```text
CPU Cache
   ↕
Coherent Interconnect
   ↕
DMA
```

如果 DMA 发：

```text
AxCACHE  = Cacheable
AxDOMAIN = Inner/Outer Shareable
```

那么：

- `AxCACHE` 告诉系统：这类访问允许按相应 cache policy 处理；
- `AxDOMAIN` 告诉 coherent fabric：需要在哪个范围内 snoop / maintain coherency。

如果改成：

```text
AxCACHE = Non-cacheable
```

意思是数据本身不按普通 cacheable memory 处理。

但即使这样，`AxDOMAIN` 仍是另一个独立维度，不能简单由 `AxCACHE` 推出来。

---

### 5. 和你前面学的 Inner/Outer 属性对应起来

可以这样整理：

```text
Cacheability
├ Inner Cacheability
└ Outer Cacheability
      ↓
关注“怎么缓存”
```

而：

```text
Shareability
├ Inner Shareable
└ Outer Shareable
      ↓
关注“一致性覆盖哪些节点”
```

`AxCACHE` 更接近前者的 transaction-side 表达，`AxDOMAIN` 则是后者的 transaction-side 表达。

### 一句话记忆

> **`AxCACHE` 管 cache policy，`AxDOMAIN` 管 coherency scope。**

或者更形象一点：

> **AxCACHE 问“怎么存”，AxDOMAIN 问“和谁保持一致”。**

## 一句话总结

AxCACHE 与 AxDOMAIN 分别表达缓存/内存行为与共享域相关信息，具体产品还会做属性转换。

## 整理注

原答是概念入门，不是合法信号组合表。AxDOMAIN 属于 ACE/ACE-Lite 扩展，不能称为所有 AXI 接口必有信号；MMU-720AE 的属性归一化和 TRM 表格差异见正式知识文档。

## 正式知识去向

本轮查证与边界说明见 [对应正式文档](../../02_KNOWLEDGE/Interfaces/08_AxCACHE_AxDOMAIN与属性转换边界.md)；原回答保留用于追溯，不以收录代替验证。
