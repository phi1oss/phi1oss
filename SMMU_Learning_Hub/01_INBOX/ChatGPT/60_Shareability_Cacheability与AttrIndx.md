# Shareability、Cacheability 与 AttrIndx

记录日期：2026-09-10
记录端：本地

来源：[SMMU](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)。以下保留原回答，去除不可解析的原引用标记；查证后的边界以关联正式知识为准。

## 问题

解析一下培训文档 16 页到 17 页中关于 shareability 和 cacheability 的内容，这两个属性的区别是什么，为什么都有 inner 和 outer 的划分

问答日期：2026-09-09；原问答标识：`bbb212ce-f364-4ba4-8cac-79b2043a0364`。

## 回答

可以把 16–17 页放在一起理解，因为这两页其实是在回答 **两个完全不同的问题**：

> **Shareability：这份数据需要在哪些 Observer 之间保持一致性？**  
> **Cacheability：这份数据在不同层次的 Cache 中允许怎么缓存？**

培训文档第 16 页把 Shareability 分成 Non-shareable、Inner Shareable、Outer Shareable；第 17 页则强调 Arm **不向软件暴露具体 Cache 物理拓扑，而是用行为描述 Cache，并允许分别设置 Inner / Outer cache policy**。

### 1. Shareability 管的是“谁要一起保持一致”

假设系统概念上是：

```text
Core0 ─┐
Core1 ─┤ Processor 0
       │
Core2 ─┐
Core3 ─┤ Processor 1
       │
      Mali GPU
```

对于某块内存：

```text
Non-shareable
→ 只认为当前 Observer 使用

Inner Shareable
→ 在一个较小的 Shareability Domain 内共享

Outer Shareable
→ 在更大的 Shareability Domain 内共享
```

所以 Shareability 的核心对象其实是 **Observer 集合**：

```text
           Outer Shareable Domain
┌────────────────────────────────────┐
│                                    │
│     Inner Shareable Domain         │
│   ┌───────────────────────┐        │
│   │ Core / Processor ...  │        │
│   └───────────────────────┘        │
│                         GPU ...     │
└────────────────────────────────────┘
```

它告诉系统：

> **哪些 Observer 的 cache 必须对这块数据表现得像“透明的一致内存”。**

Slide 也明确说，这需要 system 提供 coherency management。

---

### 2. Cacheability 管的是“怎么缓存”

Cacheability 则完全不是在问谁共享，而是在问：

> **这个 memory region 在 Cache hierarchy 中采用什么缓存策略？**

例如：

```text
Inner:
  Non-cacheable
  Write-Through
  Write-Back
  Read/Write Allocate ...

Outer:
  Non-cacheable
  Write-Through
  Write-Back
  ...
```

培训文档强调：

> Arm Architecture 是按 **cache behaviour** 描述 Inner/Outer，而不是告诉软件“L1 是 Inner、L3 是 Outer”。

所以像：

```text
Inner = WB
Outer = NC
```

真正表达的是：

> **Inner cache level 按 Write-Back Cacheable 行为处理，而 Outer cache level 按 Non-cacheable 行为处理。**

不能绝对翻译成：

```text
L1/L2 = WB
L3 = NC
```

具体物理 Cache 怎样对应 Inner/Outer，是 SoC 实现决定的。

---

### 3. 为什么 Shareability 和 Cacheability 都有 Inner / Outer？

这是你这个问题真正的关键。

因为 Arm 把 Memory System 抽象成 **嵌套的层次/域**：

```text
              Outer
     ┌─────────────────────┐
     │                     │
     │       Inner         │
     │   ┌─────────────┐   │
     │   │             │   │
     │   └─────────────┘   │
     │                     │
     └─────────────────────┘
```

但同样叫 Inner / Outer，描述的是 **两个正交维度**。

### Shareability 的 Inner / Outer

回答：

```text
Inner Shareable
→ 在较小Observer集合里保持一致

Outer Shareable
→ 在更大Observer集合里保持一致
```

关键词是：

> **Scope / 谁参与**

### Cacheability 的 Inner / Outer

回答：

```text
Inner Cacheability
→ Inner cache level采用什么cache policy

Outer Cacheability
→ Outer cache level采用什么cache policy
```

关键词是：

> **Policy / 怎么缓存**

所以千万不要理解成：

```text
Inner Shareable = Inner Cacheable
Outer Shareable = Outer Cacheable
```

完全不是一回事。

---

### 4. 用一个例子最容易看出来

假设一块 Normal Memory：

```text
Shareability:
Inner Shareable

Cacheability:
Inner = WB
Outer = WB
```

它表达了两件独立的事情：

```text
Inner Shareable
→ 指定 coherency scope
→ 哪些 Observer 必须看到一致的数据

Inner WB / Outer WB
→ 指定 cache policy
→ 不同 cache level 怎么缓存
```

甚至可以出现：

```text
Inner Shareable
+
Inner WB
+
Outer NC
```

没有矛盾。

因为：

> **“谁之间保持一致”** 和 **“这些层次是否缓存、怎么缓存”** 本来就是两个问题。

---

### 最适合面试的表述

| 属性 | 回答的问题 | Inner / Outer 的含义 |
|---|---|---|
| **Shareability** | 谁需要参与一致性？ | 较小 / 较大的 Observer scope |
| **Cacheability** | 数据怎么缓存？ | Inner / Outer cache level 的 policy |

所以一句话记忆：

> **Shareability 的 Inner/Outer 划的是“人群范围”；Cacheability 的 Inner/Outer 划的是“Cache 行为层次”。**

而培训文档这两页放在一起，其实就是希望你建立这个边界：

> **Shareability 决定 coherency scope，Cacheability 决定 caching policy；二者都采用 Inner/Outer 的层次抽象，但解决的是完全不同的问题。** 

---

## 问题

cacheability 的内容是否与 attrindx 的索引内容重复？二者的区别是什么

问答日期：2026-09-09；原问答标识：`bbb21748-2fb8-45ca-9d35-d02897c02d86`。

## 回答

不重复，二者是 **“索引”和“被索引的内容”** 的关系。

> **`AttrIndx` 只是页表描述符里的一个索引号；真正的 Cacheability / Memory Type 定义在 `MAIR_ELx` 对应的属性项里。**

最小关系可以记成：

```text
PTE.AttrIndx
    ↓
MAIR_ELx[AttrIndx]
    ↓
Memory Attribute
├ Device / Normal
├ Inner Cacheability
└ Outer Cacheability
```

例如：

```text
PTE.AttrIndx = 3
```

表示：

> “这页使用 MAIR 中第 3 号 memory attribute。”

假设：

```text
MAIR[3]:
Outer = Write-Back
Inner = Write-Back
```

那么这个 page 最终就是：

```text
Inner WB
Outer WB
```

所以：

- `AttrIndx`：**选哪一套属性**
- `MAIR` entry：**这套属性具体是什么**
- Cacheability：**MAIR entry 里面真正定义的缓存策略**

另外要注意：

> **Shareability 不在 MAIR/AttrIndx 里。**

它通常由页表描述符中的 `SH` 等字段单独描述：

```text
PTE
├ AttrIndx → MAIR → Inner/Outer Cacheability
└ SH       → Inner/Outer Shareability
```

### 一句话记忆

> **`AttrIndx` 是“菜单编号”，MAIR 是“菜单内容”，Inner/Outer Cacheability 是菜单里具体写的缓存策略。**

## 一句话总结

Shareability 描述共享范围，Cacheability 描述缓存策略；AttrIndx 选择 MAIR 中的属性项。

## 整理注

保留原答。Non-shareable 不是禁止其他设备访问的权限设置；Inner/Outer domain 与具体 cluster/cache 层级并非固定对应。

## 正式知识去向

本轮查证与边界说明见 [对应正式文档](../../02_KNOWLEDGE/Translation/03_Translation_Descriptors与Memory_Attributes.md)；原回答保留用于追溯，不以收录代替验证。
