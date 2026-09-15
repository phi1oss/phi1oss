# AArch64 多级页表与地址翻译

> 整理日期：2026-08-30  
> 状态：正式知识文档，待手工 Walk 验证

## 核心结论

**AArch64 使用多级 translation table，把 VA 的不同 bitfield 依次作为各级表的 index；walk 遇到 Table descriptor 就进入下一级，遇到合法 Block/Page descriptor 就得到输出地址基址和属性，再与 VA offset 组合形成最终 PA。**

多级结构的目的，是让软件只为实际使用的地址范围分配下级页表，而不必为整个巨大 VA space 建立一张完全展开的单级表。

## 1. 4KB Granule 下的典型 48-bit VA

对于 4KB granule、48-bit VA 的典型四级结构：

```text
VA[47:39]  → Level 0 index
VA[38:30]  → Level 1 index
VA[29:21]  → Level 2 index
VA[20:12]  → Level 3 index
VA[11:0]   → 4KB page offset
```

每个 table entry 为 8 bytes。一次 walk 可概念化为：

```text
TTBR gives L0 table base
        ↓
L0 entry address = L0 base + VA[47:39] × 8
        ↓ Table descriptor
L1 entry address = L1 base + VA[38:30] × 8
        ↓ Table descriptor or 1GB Block
L2 entry address = L2 base + VA[29:21] × 8
        ↓ Table descriptor or 2MB Block
L3 entry address = L3 base + VA[20:12] × 8
        ↓ 4KB Page descriptor
PA = Output Page Base + VA[11:0]
```

## 2. Table、Block、Page 与 Fault Descriptor

- **Table descriptor**：给出下一级 translation table 的地址，walk 继续。
- **Block descriptor**：在允许的中间 level 直接形成较大范围的最终 mapping。
- **Page descriptor**：在最后一级形成基础 granule 大小的最终 mapping。
- **Fault/Invalid descriptor**：不存在有效 mapping，walk 终止并产生 Translation Fault。

因此，translation granule 与实际 mapping size 不相同：

```text
4KB granule
├─ L3 Page → 4KB mapping
├─ L2 Block → 2MB mapping
└─ L1 Block → 1GB mapping
```

## 3. 起始 Level 由什么决定？

并非所有 translation regime 都必须从 L0 开始。起始 lookup level 由以下因素共同决定：

- TCR 中配置的输入地址范围，例如 `TnSZ`。
- Translation granule：4KB、16KB 或 64KB。
- 实现支持的 VA/PA range。

地址空间较小时，高层 index bit 不存在，因此 walk 可以直接从较低 level 开始。

## 4. 三种 Translation Granule

Training 材料给出的典型关系是：

- **4KB**：48-bit 地址时常见四级 lookup，每级 index 9 bit。
- **16KB**：48-bit 地址时可使用四级 lookup，每级主要 index 11 bit。
- **64KB**：48-bit 地址时常见三级 lookup，每级主要 index 13 bit。

Granule 决定最小 page size、每张表的 entry 数量和 VA bit 的切分方式，但不意味着所有 mapping 或 TLB entry 都只能覆盖一个 granule。

## 5. 手工 Walk 的标准步骤

面对给定 VA、TTBR 和内存 dump 时：

1. 根据 TCR 确认 VA size、granule 和起始 level。
2. 按 granule 拆分 VA 的各级 index 与最终 offset。
3. 使用 `table_base + index × 8` 计算当前 descriptor 地址。
4. 解码 descriptor 类型和 Valid 状态。
5. Table descriptor：取出 next-level table address 并继续。
6. Block/Page descriptor：取出 output base 和 attributes，停止 walk。
7. 根据 mapping size 保留相应 VA offset，组合最终 PA。
8. 检查地址宽度、权限、AF、memory attributes 等，确认访问是否真正允许。

## 6. 常见错误

- 把 VA 的十六进制数字直接当作各级 index，没有先按 bitfield 拆分。
- 忘记每个 descriptor 是 8 bytes，entry address 没有乘 8。
- 遇到 Block descriptor 后仍继续进入下一级。
- 使用 4KB page 的 12-bit offset 去计算 2MB Block translation。
- 只计算 PA，不检查 descriptor attributes 和 fault 条件。
- 忽略 TCR 配置，默认所有地址都从 L0 开始。

## 一句话总结

> **多级页表 Walk 就是用 VA 的各级 index 逐层读取 descriptor；Table descriptor 继续走，Block/Page descriptor 给出结果，最终用 output base 加上与 mapping size 对应的 VA offset 得到 PA。**

## 来源与相关资料

- [MMU.pdf](../../99_SOURCE/Training/MMU.pdf)：PDF 第 5–10、29–31 页，多级页表、手工 walk、granule、TLB 与 BBM。
- 相关正式知识：[Translation Descriptors 与 Memory Attributes](03_Translation_Descriptors与Memory_Attributes.md)。
