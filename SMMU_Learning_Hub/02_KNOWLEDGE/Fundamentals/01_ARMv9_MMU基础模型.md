# ARMv9 MMU 基础模型

> 整理日期：2026-08-30  
> 更新日期：2026-09-11  
> 状态：正式知识文档；首测完成，L2 已验证，L3 待复测
> 范围：AArch64/Armv9 处理器侧 Stage 1 Memory Management 基础

## 掌握验证

2026-09-11 归档首轮 10 次作答：**L2 已验证，具备部分 L3 表现，L3 待独立复测。** 已能区分翻译/数据缓存、判断权限限制、解释离散物理页映射并完成页偏移计算。

重点复测：页表入口与 ASID 的分工、旧 TLB 与新页表的可能状态、含命中/未命中及失败分支的完整 load 流程。提示后答对与独立答对已分别保留。

- [完整测试记录：MMU 基础模型首测](../../04_INTERVIEW/Assessments/2026-09-11_MMU基础模型首测.md)

结果仅对应本篇基础模型，不代表完整多级 walk、全部 regime、维护/BBM 或 Fault 专题已通过。

## 核心结论

**MMU 的核心职责，是根据当前 translation regime 把 PE 发出的虚拟地址转换成物理地址，同时给出访问权限、执行权限、memory type、cacheability 和 shareability 等属性。**

MMU 不只是一个 `VA → PA` 计算器。它把软件看到的地址空间与系统实际物理布局解耦，并在同一条 translation 路径上完成隔离和属性检查。

## 1. 为什么需要 MMU？

如果软件直接使用系统物理地址，软件将被具体 DRAM、Flash 和 MMIO 布局绑定，也很难让不同进程拥有彼此隔离的地址空间。

MMU 提供的价值包括：

- 隐藏物理内存的碎片和实际布局。
- 为不同进程或软件层建立独立虚拟地址空间。
- 控制读、写、执行和特权访问。
- 为每个区域指定 Normal/Device、cacheability、shareability 等 memory attributes。
- 在虚拟化场景中支持 `VA → IPA → PA` 两阶段转换。

## 2. VA、PA 与 Translation Result

开启地址转换后，PE 产生 Virtual Address。MMU 根据 translation table 找到输出地址基址，并保留 VA 的低位 offset：

```text
VA
├─ Translation Table Index bits
└─ Offset bits

Translation Table Walk
        ↓
PA Base + Attributes

Final PA = PA Base + VA Offset
```

最终 translation result 不只有 PA，还包括：

```text
Output PA / IPA
Mapping size
Read / Write permission
Execute permission
Memory type and cacheability
Shareability
Security and translation-context information
```

## 3. TLB 与 Table Walk 的分工

TLB 是 translation cache，缓存已经解析完成的 translation result。

```text
Memory Access with VA
        ↓
      TLB Lookup
        ├─ Hit  → 使用 PA、权限和属性
        └─ Miss → Translation Table Walk
                        ↓
                   得到 leaf descriptor
                        ↓
                   解析并填充 TLB
                        ↓
                   继续原访问
```

TLB 缓存的不是“PTE 在哪里”，而是“PTE 和相关寄存器已经解析后的结果”。因此软件修改页表后，即使内存中的 descriptor 已经改变，处理器仍可能命中旧 TLB entry；这就是必须进行 TLB maintenance 的根本原因。

## 4. MMU 同时承担访问控制

一次 translation 成功找到 PA，并不意味着访问一定被允许。MMU 还会检查：

- 当前访问是 Read、Write 还是 Execute。
- 当前 Exception Level 是否具有权限。
- descriptor 是否设置 UXN/PXN。
- Stage 1 或 Stage 2 access permission 是否允许。
- Access Flag 等状态是否满足。

所以：

```text
TLB Hit
≠ Access Automatically Allowed
```

TLB hit 只表示找到了 cached translation，随后仍要使用缓存的权限和属性完成检查。

## 5. 与 SMMU 的关系和边界

CPU MMU 与 SMMU 使用相同的页表、TLB、Stage 1/Stage 2 等基本思想，但请求来源不同：

```text
CPU MMU
→ 为 PE 执行的 load/store/fetch 做地址转换

SMMU
→ 为设备 DMA 请求做地址转换和隔离
```

当前文档只建立处理器侧 Memory Management 基础。StreamID、STE、CD、TBU、TCU 和 DTI 属于后续 SMMU 实现层，不应提前混入 CPU translation regime 的定义。

## 易混淆点

- **页表不是 MMU。** 页表是软件维护的数据结构，MMU 是读取、解释并缓存其结果的硬件机制。
- **TLB 不是普通数据 Cache。** Cache 缓存数据或指令，TLB 缓存 translation result。
- **映射存在不等于访问合法。** 权限和属性检查仍可能产生 fault。
- **关闭 translation 时的地址行为不应简单描述为“所有软件地址永远都是虚拟地址”。** 当前 translation regime 是否启用由 SCTLR 等控制。

## 一句话总结

> **MMU 根据当前 translation regime 把 VA 转换为 PA，并同时给出权限与 memory attributes；TLB 负责缓存结果，miss 时才进行 table walk。**

## 来源与相关资料

- [MMU.pdf](../../99_SOURCE/Training/MMU.pdf)：PDF 第 2–6 页，MMU 目的、虚拟地址、VA→PA 与多级翻译基础。
- 后续相关正式知识：[AArch64 多级页表与地址翻译](../Translation/01_AArch64多级页表与地址翻译.md)。
