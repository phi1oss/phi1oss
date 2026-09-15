# Linear 与 2-Level Stream Table

> 记录日期：2026-09-01  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 原始上下文：两页培训视频截图，时间点未知  
> 状态：Inbox 知识点整理；`SPLIT`、`Span` 及地址计算的精确编码待结合 SMMUv3 Architecture Specification 校验

## 凝练问题

**两页培训材料分别介绍了哪些知识点？SMMUv3 的 Stream Table/STE 保存什么信息，Linear 与 2-Level Stream Table 如何使用 StreamID 找到 STE，为什么需要两级结构？**

## 核心结论

> **第一页说明 Stream Table 是 `StreamID → STE → Translation Context` 的配置入口；第二页说明当 StreamID 空间很大且使用稀疏时，可以把 Stream Table 组织为两级结构，由 SID 高位索引 L1 Descriptor、L1 指向按需分配的 L2 Table，再由 SID 低位选择真正的 STE。**

面试式一句话：

> **SMMUv3 的 Stream Table 可以采用 Linear 或 2-Level 格式。Linear 模式用 StreamID 直接索引固定 64B 的 STE，结构简单但可能需要为大范围 SID 预留连续内存；2-Level 模式把 SID 拆成高低两部分，高位选择 L1 Descriptor 和 L2 Table，低位选择 L2 中的 STE，从而更适合稀疏的 StreamID 空间。**

# 第一页：Stream Table 与 STE

## 1. StreamID 用来选择 STE

SMMU 收到一笔 Transaction 后，使用其中的 `StreamID` 查找 Stream Table 中相应的 STE：

```text
Device Transaction
      │
      ├─ Address
      └─ StreamID
            ↓
       Stream Table
            ↓
           STE
```

Stream Table 不是地址映射页表，而是：

> **按 StreamID 组织的 Per-stream Configuration Structure。**

真正的地址映射仍保存在 Stage 1/Stage 2 Translation Table 中。

## 2. STE 保存 Stream 的 Translation Configuration

一个 STE 固定为 64 Bytes，主要回答：

```text
这个 Stream 配置是否有效？
采用 Bypass、Stage 1、Stage 2 还是 S1+S2？
Stage 2 Translation Context 是什么？
Stage 1 Context Descriptor 在哪里？
如何解释该 Stream 的 Translation World？
是否涉及 Substream？
```

可以概括为：

```text
STE
├─ Stream 有效性与总体模式
│  ├─ V / Valid
│  ├─ Config
│  └─ STRW
│
├─ Stage 2 Translation Context
│  ├─ VMID
│  ├─ S2TTB
│  └─ 其他 Stage 2 参数
│
└─ Stage 1 入口
   └─ S1ContextPtr
      → CD 或 CD Table
```

因此：

> **STE 不是 Page-table Entry，而是 Stream 级 Translation Configuration 的入口。**

## 3. 关键字段的概念含义

### `V` / Valid

表示该 StreamID 是否拥有可用的 Stream Configuration：

```text
V = 1 → STE 配置有效
V = 0 → 没有有效的 Stream Configuration
```

### `Config`

决定该 Stream 使用哪种 Translation Mode：

```text
Full Bypass
Stage 1 only
Stage 2 only
Stage 1 + Stage 2
```

### `VMID` 与 `S2TTB`

Stage 2 Translation Context 直接由 STE 描述：

```text
STE.VMID  → 标识 Stage 2/VM Translation Context
STE.S2TTB → Stage 2 Translation Table Base
```

概念路径：

```text
IPA
 ↓
STE.S2TTB + VMID
 ↓
Stage 2 Translation Table
 ↓
PA
```

### `S1ContextPtr`

Stage 1 配置不是全部直接放在 STE 中，而是由 STE 指向 CD 或 CD Table：

```text
SID
 ↓
STE
 ↓ S1ContextPtr
CD / CD Table
 ↑
SSID 选择具体 CD（Substream 场景）
 ↓
Stage 1 Translation Table
```

因此最简层级是：

> **SID 选择 STE；STE 管 Stream 总体模式和 Stage 2；SSID 选择 CD；CD 管 Stage 1。**

### `STRW`

`STRW` 描述 Stream World，即这套 Stream Translation Context 按何种架构 Translation World/Execution Model 解释。它与 `Config` 的区别是：

```text
Config → 采用哪一级 Translation
STRW   → 按哪种 Architectural World 解释 Context
```

具体编码应以 SMMUv3 Architecture Specification 为准。

## 4. Linear Stream Table

最直接的组织方式是所有 STE 在线性、连续的 Stream Table 中排列：

```text
SMMU_STRTAB_BASE
       ↓
+------------------+
| STE[0]     64B   |
| STE[1]     64B   |
| STE[2]     64B   |
| ...              |
| STE[N]     64B   |
+------------------+
```

概念上的查找关系为：

```text
STE_Address ≈ STRTAB_BASE + StreamID × 64B
```

这里的公式用于说明直接索引思想；实际地址形成、支持的 SID 位宽、对齐和边界必须按架构配置解释。

Linear 的主要优点是：

> **结构简单，StreamID 可以直接定位 STE。**

# 第二页：2-Level Stream Table

## 5. Linear Table 的稀疏空间问题

如果 StreamID Width 很大，理论 SID 数量可能远大于系统实际使用的 Stream 数量。

以 16-bit SID 为例：

```text
SID 数量 = 2^16 = 65536
STE 大小 = 64 Bytes
完整 Linear Table = 65536 × 64B = 4 MiB
```

即便系统只使用少量且分布很散的 SID，Linear Table 仍可能需要为较大的 SID Namespace 准备连续空间。

本质问题是：

> **StreamID Namespace 可能很大而且稀疏，但 Linear Table 按配置覆盖范围直接展开 STE。**

## 6. 2-Level Stream Table 的结构

两级结构不再让 `STRTAB_BASE` 直接指向所有 STE，而是：

```text
STRTAB_BASE
     ↓
  L1 Table
     ↓
L1 Descriptor
     ↓ L2Ptr
  L2 Table
     ↓
    STE
```

L1 Entry 本身不是 STE，其主要状态是：

```text
Invalid
或
Pointer to an L2 Table
```

真正保存 `V`、`Config`、`VMID`、`S2TTB`、`S1ContextPtr` 等字段的 STE 位于 L2 Table。

## 7. StreamID 高低位分工

在两级格式中，StreamID 被拆成两部分：

```text
StreamID
┌────────────────┬────────────────┐
│ High SID bits  │ Low SID bits   │
└────────────────┴────────────────┘
        ↓                 ↓
    L1 Index           L2 Index
```

完整查找路径是：

```text
StreamID 高位
     ↓
选择 L1 Descriptor
     ↓
读取 L2Ptr
     ↓
定位 L2 Table
     ↓
StreamID 低位选择 L2 中的 STE
     ↓
获得该 Stream 的 Translation Configuration
```

一句话记忆：

> **SID 高位找“哪张 L2”，SID 低位找“L2 中的哪个 STE”。**

## 8. `SPLIT`、`L2Ptr` 与 `Span`

### `SMMU_STRTAB_BASE_CFG.SPLIT`

`SPLIT` 决定 SID 的切分位置，即哪些低位用于 L2 Index、剩余高位用于 L1 Index。概念上，如果低 `n` 位用于 L2，则一个完整 L2 区域最多对应 `2^n` 个 SID/STE。

因此：

> **`SPLIT` 描述整张 2-Level Stream Table 如何划分 SID Namespace。**

### `L2Ptr`

L1 Descriptor 中的 `L2Ptr` 指向对应 L2 Stream Table 在内存中的位置。

### `Span`

培训图中的 `Span[4:0]` 用来描述该 L1 Descriptor 所指 L2 Structure 的实际覆盖范围或容量。

当前阶段可先区分：

```text
SPLIT → 全局决定 SID 高低位如何切分
Span  → 描述某个 L1 Descriptor 指向的 L2 区域规模
```

`Span` 的精确编码、合法范围及其与 Leaf Size 的关系需要按 SMMUv3 Architecture Specification 继续校验，不把当前概念解释当成最终字段定义。

## 9. 为什么两级结构可以节省内存

当某段 SID 范围没有使用时：

```text
对应 L1 Entry = Invalid
          ↓
不分配相应 L2 Table
```

只有真正使用的 SID Region 才按需分配 L2：

```text
Large + Sparse SID Namespace
             ↓
        Small L1 Table
             ↓
     L2 Tables allocated on demand
```

因此 2-Level 最大的工程价值是：

> **避免为未使用的 SID Region 分配完整 STE 空间。**

这属于从架构结构得到的合理设计目的解释；精确内存收益取决于 SID Width、`SPLIT`、已分配 L2 的数量和实现约束。

## 10. Linear 与 2-Level 对比

| 对比项 | Linear Stream Table | 2-Level Stream Table |
|---|---|---|
| SID 查找方式 | SID 直接索引 STE | SID 高位索引 L1，低位索引 L2 中的 STE |
| 层级 | 一层 | L1 + L2 |
| STE 位置 | Linear Table | L2 Table |
| L1 内容 | 无 | Invalid 或 L2 Pointer/Descriptor |
| 主要优点 | 简单、直接 | 适合大而稀疏的 SID Namespace，L2 可按需分配 |
| 主要代价 | 稀疏空间可能浪费内存，并受连续分配要求影响 | 配置、查找和维护更复杂 |

## 11. 与 CPU 多级页表的类比及边界

二者的共同设计思想是：

> **面对巨大且稀疏的索引空间，使用多级结构按需分配下级表。**

```text
CPU Page Table
VA → 多级 Table → PTE

SMMU 2-Level Stream Table
StreamID → L1 Descriptor → L2 Table → STE
```

但两者索引和结果不同：

```text
CPU Page Table：VA → 地址映射 PTE
Stream Table：SID → Stream Configuration STE
```

因此，Stream Table 不是 Translation Table；它是找到 Translation Context 的配置入口。

## 面试式回答

> **Linear Stream Table 直接使用 StreamID 索引固定大小的 STE，结构简单，但当 StreamID Width 很大而实际 Stream 很稀疏时，可能需要为大量未使用 SID 预留连续内存。2-Level Stream Table 把 StreamID 拆成高位和低位，高位索引 L1 Descriptor，L1 通过 `L2Ptr` 指向按需分配的 L2 Table，低位再选择其中的 STE。`STRTAB_BASE_CFG.SPLIT` 决定 SID 的切分方式；未使用的 SID Region 可以保持 L1 Invalid、不分配 L2，从而降低 Stream Table 的内存占用。**

## 一句话总结

> **第一页回答“STE 是什么、里面配置什么”；第二页回答“当 SID 空间很大且稀疏时，如何用 L1/L2 更高效地组织这些 STE”。**

## 相关记录与资料

- [StreamID 与 Translation Context 映射](18_StreamID与Translation_Context映射.md)
- [Substream、SSID 与 PCIe PASID](19_Substream_SSID与PASID.md)
- [SMMUv3 Programming Model](23_SMMUv3_Programming_Model.md)
- [SMMUv3 寄存器空间与 Memory-based 配置](26_SMMUv3寄存器空间与Memory-based配置.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

