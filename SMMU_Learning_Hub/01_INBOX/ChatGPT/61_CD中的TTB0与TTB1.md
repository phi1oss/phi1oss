# CD 中的 TTB0 与 TTB1

记录日期：2026-09-10
记录端：本地

来源：[SMMU](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)。以下保留原回答，去除不可解析的原引用标记；查证后的边界以关联正式知识为准。

## 问题

CD 数据结构中的 TTB0 和 TTB1 分别是什么字段

问答日期：2026-09-09；原问答标识：`bbb21b2c-59fd-427d-9b80-94e9c3152738`。

## 回答

在 SMMUv3 的 **Context Descriptor（CD）** 里：

> **TTB0 和 TTB1 都是 Stage 1 translation table 的基地址字段。**
>
> `TTB0` 指向 **Translation Table 0** 的基地址，`TTB1` 指向 **Translation Table 1** 的基地址。

可以类比 CPU 的：

```text
TTBR0_EL1
TTBR1_EL1
```

---

### 1. TTB0

`TTB0`：

> **Stage 1 Translation Table 0 的根页表基地址。**

它和这些字段配套：

```text
T0SZ   → TTB0覆盖的地址空间大小
TG0    → TTB0使用的translation granule
IR0/OR0/SH0
       → 对TTB0进行table walk时的cacheability/shareability
EPD0   → 是否禁止TTB0 table walk
```

培训文档也明确列出：

```text
TTB0 = Address of TT0 base
```



典型路径：

```text
SSID
 ↓
CD
 ↓
TTB0
 ↓
Stage1 page table
 ↓
VA/IOVA → IPA/PA
```

---

### 2. TTB1

`TTB1`：

> **Stage 1 Translation Table 1 的根页表基地址。**

配套字段类似：

```text
T1SZ
TG1
IR1/OR1/SH1
EPD1
TTB1
```

培训文档列的是：

```text
TTB1 = Address of TT1 base
```



但 `TTB1` 并不是所有场景都必须使用。MMU-720AE TRM 特别说：

> 如果当前 StreamWorld 支持 split address spaces，可以通过 `EPD1` 控制是否启用 TTB1 的 table walk。

---

### 3. 为什么一个 CD 需要两个 TTB？

因为一个 Stage 1 address space 可以被划分成两个 VA region，各自由一棵页表负责：

```text
           Stage1 address space

        region 0             region 1
           │                    │
          TTB0                 TTB1
           │                    │
      Page Table 0         Page Table 1
```

这和 AArch64 CPU 使用：

```text
TTBR0_EL1
TTBR1_EL1
```

的思路非常接近。

CPU 常见使用方式是：

```text
TTBR0_EL1 → 用户地址空间
TTBR1_EL1 → 内核地址空间
```

但在 SMMU 中，不要机械理解成：

```text
TTB0 = user
TTB1 = kernel
```

更准确地说：

> **TTB0/TTB1 是同一个 Stage 1 translation context 下两套 VA-region translation-table root。具体如何使用由 translation regime / StreamWorld 配置决定。**

---

### 4. 和 STE 里的 `S2TTB` 不要混

这是最重要的边界：

```text
CD.TTB0 / CD.TTB1
→ Stage 1 translation table root
```

而：

```text
STE.S2TTB
→ Stage 2 translation table root
```

所以：

```text
SSID
 ↓
CD
 ├─ TTB0
 └─ TTB1
      ↓
    Stage1
      ↓
     IPA

SID
 ↓
STE
 ↓
S2TTB
 ↓
Stage2
 ↓
 PA
```

MMU-720AE TRM 也明确说 CD 定义 Stage 1 translation，而 STE 配置 Stage 2 translation 及 CD table 的定位。

### 一句话记忆

> **CD 里的 TTB0/TTB1 就是 SMMU Stage 1 的两棵页表根；STE 里的 S2TTB 才是 Stage 2 页表根。**

## 一句话总结

CD.TTB0/TTB1 是 Stage 1 的两个页表根；STE.S2TTB 才是 Stage 2 页表根。

## 整理注

只收录带完整回答的一次问答，略过同文无回答的重复用户消息。

## 正式知识去向

本轮查证与边界说明见 [Programming the SMMU：配置选择主线](../../02_KNOWLEDGE/Software/02_Programming_the_SMMU主线.md#configuration-lookup)；原回答保留用于追溯，不以收录代替验证。
