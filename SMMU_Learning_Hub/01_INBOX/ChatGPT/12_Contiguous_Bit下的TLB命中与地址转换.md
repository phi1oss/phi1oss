# Contiguous Bit 下的 TLB 命中与 VA→PA 转换

## 问题

假设一次 table walk 得到了带 `Contiguous=1` 的页表项，之后继续访问同一 contiguous group 中的下一个 page 时，硬件如何找到对应的 TLB entry，并得到 VA→PA translation result？

## 核心结论

**如果 TLB 选择利用 Contiguous Bit 做合并，那么第一次 table walk 发现 `Contiguous=1` 后，可以把整个 contiguous group 表示成一个覆盖范围更大的 cached translation。后续访问组内其他 page 时，不需要查找“下一个 PTE”，而是直接命中这个 enlarged TLB translation，再用 VA 相对于 group base 的 offset 计算 PA。**

```text
PA = PA_group_base + (VA - VA_group_base)
```

Contiguous Bit 的优化发生在 **TLB translation coverage** 上，并不意味着页表中的多个 PTE 变成了一个大 PTE。

## 1. 第一次访问：Table Walk 发现 Contiguous Group

以 4KB translation granule、16 个 contiguous page 为例：

```text
VA                      PA
0x4000_0000  ───────►  0x8000_0000
0x4000_1000  ───────►  0x8000_1000
0x4000_2000  ───────►  0x8000_2000
    ...                    ...
0x4000_F000  ───────►  0x8000_F000

16 × 4KB = 64KB
```

这 16 个 PTE 满足：

```text
Contiguous = 1
VA 连续
PA 连续
相关属性一致
处在相同 Translation Level
Group Base 满足 64KB 对齐要求
```

第一次访问：

```text
VA = 0x4000_3000
```

TLB miss 后执行 table walk：

```text
VA 0x4000_3000
      ↓
   TLB Miss
      ↓
  Table Walk
      ↓
找到对应 PTE
PA Base = 0x8000_3000
Contiguous = 1
```

硬件由 descriptor 和 translation regime 得知，该映射属于一个 16×4KB 的 contiguous group，因此可以形成概念上的 enlarged translation：

```text
TLB Translation

VA Base : 0x4000_0000
PA Base : 0x8000_0000
Size    : 64KB
Attr    : ……
```

这里的 `64KB` 是 cached translation 的 coverage，而不是页表突然出现了一个 64KB PTE。

## 2. 后续访问：按范围命中同一个 Translation

之后访问组内另一个 page：

```text
VA = 0x4000_7000
```

TLB lookup 不需要重新寻找该 VA 对应的 4KB PTE，而是检查它是否落在已经缓存的 VA range 中：

```text
Cached VA Range

0x4000_0000 ～ 0x4000_FFFF
```

因为：

```text
0x4000_7000 ∈ [0x4000_0000, 0x4000_FFFF]
```

所以命中这个 64KB translation。

然后计算 group offset：

```text
VA Group Offset
= VA - VA_group_base

= 0x4000_7000 - 0x4000_0000
= 0x7000
```

再组合得到 PA：

```text
PA
= PA_group_base + VA_group_offset

= 0x8000_0000 + 0x7000
= 0x8000_7000
```

完整路径是：

```text
VA = 0x4000_7000
        ↓
     TLB Lookup
        ↓
命中 64KB Contiguous Translation
        ↓
Group Offset = 0x7000
        ↓
PA Base + Group Offset
        ↓
PA = 0x8000_7000
```

**整个过程不需要再次执行 table walk。**

## 3. TLB 如何处理不同的 Translation Coverage？

同一个 TLB 中，不同 cached translation 的覆盖范围可以不同：

```text
Entry A → Coverage = 4KB

Entry B → Coverage = 64KB
          Contiguous Group

Entry C → Coverage = 2MB
          Block Descriptor
```

因此，lookup 时不同 entry 的 tag/offset 边界可以不同。

### 4KB Translation

```text
VA
|          Tag          | Offset |
                         12 bits
```

低 12 bit 表示 4KB page 内 offset。

### 64KB Contiguous Translation

```text
VA
|       VA Tag       | VA[15:0] |
          ↓               ↓
       TLB Match       Group Offset
```

对 64KB coverage 来说，低 16 bit 是 group 内 offset，VA 的 bit[15] 以上部分用于匹配 group base。

PA 可以概念化为：

```text
PA
|       PA Base      | VA[15:0] |
```

所以 enlarged translation 的核心是：

> **使用更大范围的 tag match，并保留更多低位 VA bit 作为 range offset。**

## 4. Contiguous Bit 不等于强制合并成一条物理 Entry

需要明确区分架构效果和微架构实现。

Contiguous Bit 表示：

> **硬件被允许对一组连续 translation 做更大范围的 TLB caching 优化。**

它不规定所有 Arm TLB 必须把多个 PTE 物理合并为一条 SRAM entry。

可能的实现包括：

```text
实现 A
Contiguous Group
      ↓
合并为一个 64KB TLB Entry
      ↓
组内后续访问直接命中
```

```text
实现 B
Contiguous Group
      ↓
内部仍使用多个 Entry 或其他组织方式
      ↓
实现架构等效的更大范围缓存优化
```

以下细节属于具体 CPU/SMMU 的微架构实现，不能仅根据 Contiguous Bit 推断：

- TLB tag 的物理格式。
- 是否真的只占用一条 SRAM entry。
- 第一次 walk 后是否读取相邻 PTE。
- 内部如何验证或存储 contiguous group 属性。

架构保证的是可观察行为和使用约束，而不是固定的 TLB 电路组织。

## 5. 完整工作流程

```text
第一次访问 Contiguous Group 中的 Page
                 ↓
              TLB Miss
                 ↓
             Table Walk
                 ↓
      发现 Descriptor 的 Contiguous=1
                 ↓
TLB 可以缓存覆盖整个 Group 的 Translation
                 ↓
以后访问同组的其他 Page
                 ↓
按 Enlarged VA Range 命中 Translation
                 ↓
计算 VA 相对于 Group Base 的 Offset
                 ↓
PA = PA_group_base + Offset
                 ↓
无需再次 Table Walk
```

## 一句话总结

> **Contiguous Bit 允许 TLB 把多个连续小页表示为一个 coverage 更大的 cached translation；组内后续 VA 通过“大范围 tag match + group offset”直接得到 PA，不需要逐页重新执行 table walk。**
