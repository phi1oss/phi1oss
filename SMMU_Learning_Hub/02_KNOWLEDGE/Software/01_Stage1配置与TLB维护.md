# Stage 1 配置与 TLB 维护

> 整理日期：2026-08-30  
> 状态：正式知识文档，待配置顺序与 BBM 练习验证

## 核心结论

**建立 Stage 1 translation regime 的本质，是让硬件同时知道“表在哪里、地址怎么拆、属性怎么解释、页表内容是什么、何时开始启用”。页表更新后必须按架构要求完成写入排序、TLB invalidation 和同步，否则硬件可能继续使用旧 translation。**

## 1. 基本配置对象

```text
TTBR_ELx
→ First-level translation table base，EL1 还可携带 ASID

TCR_ELx
→ VA size、granule、PA size、table-walk cacheability/shareability 等

MAIR_ELx
→ AttrIndx 对应的 Normal/Device memory encoding

Translation Tables
→ VA→PA mapping、permission 和 attributes

SCTLR_ELx
→ Enable MMU，并控制相关 cache/translation 行为
```

## 2. 建立 Stage 1 Regime 的概念顺序

Training 材料中的基本流程可整理为：

```text
1. 分配并初始化 translation tables
2. 配置 TTBR：first-level table base
3. 配置 MAIR：所需 memory types
4. 配置 TCR：VA/PA size、granule、walk attributes
5. 确保 translation tables 覆盖预期 VA range
6. TLBI + barrier：清除 reset/旧 context 的未知或陈旧 TLB 状态
7. 配置 SCTLR：启用 MMU，按需要启用 cache
8. ISB：使后续指令使用新的 translation state
```

实际启动代码必须满足对应 EL、Security state 和架构版本的精确顺序；上面是理解框架，不替代 Architecture Reference Manual 中的伪代码和约束。

## 3. 为什么改了 PTE 还要 TLBI？

因为 TLB 缓存的是已经解析完成的 translation result：

```text
Old PTE: VA A → PA 0x1000
        ↓ walk and fill
TLB:     VA A → PA 0x1000

Software changes PTE:
New PTE: VA A → PA 0x2000

Without TLBI:
VA A → TLB Hit → still PA 0x1000
```

内存中的 PTE 更新不会自动改写所有 PE 的 TLB entry。

## 4. 常见维护顺序

概念上的常见顺序是：

```text
Write translation table descriptor
        ↓
DSB：确保页表写入到达要求的观察范围
        ↓
TLBI：按 VA/ASID/VMID/EL 等选择合适范围
        ↓
DSB：等待 invalidation 完成
        ↓
ISB：当前 PE 后续执行使用新 translation context/state
```

具体 barrier domain、TLBI variant 和是否需要 ISB 取决于更新类型和 shareability scope，不能机械地对所有场景固定使用同一条指令。

## 5. TLBI 范围

TLBI 可以针对不同范围，例如：

- 当前 EL 的全部 translation。
- 某个 ASID。
- 某个 VA + ASID。
- 某个 VA，不区分 ASID。
- Guest Stage 1，或 Guest Stage 1 + Stage 2。
- Inner Shareable domain。

选择原则是在保证正确性的前提下尽量缩小范围，因为大范围 invalidation 会显著影响性能。

## 6. Break-Before-Make

当软件改变一个 mapping，使新旧 translation 可能同时覆盖同一 VA，特别是改变 block/page size 时，必须遵循适用的 BBM 规则：

```text
Old Valid Mapping
        ↓
Break：写 Invalid descriptor
        ↓
DSB
        ↓
TLBI
        ↓
DSB，确认旧 translation 不再可用
        ↓
Make：写入 New descriptor
        ↓
必要的同步
```

它解决的危险窗口是：

```text
Old 4KB TLB entry
+
New 2MB TLB entry
        ↓
同一 VA 同时命中不同 translation result
```

如果产生彼此冲突的 TLB entry，是否报告 TLB Conflict Abort 可以是 IMPLEMENTATION DEFINED，但软件必须遵守架构维护规则，不能依赖某个实现恰好选择其中一项。

## 7. Contiguous Group 的维护

Contiguous Bit 允许硬件扩大 group 的 TLB coverage。因此修改 group 中任一 descriptor 时，软件必须遵守架构对整个 contiguous mapping 的更新和 invalidation 要求，不能假设 TLB 只缓存被修改的单个 4KB PTE。

## 常见错误

- 只写新 descriptor，没有使旧 TLB translation 失效。
- TLBI 之前缺少确保 page-table write 可见的 barrier。
- 发出 TLBI 后没有等待其完成。
- 使用范围过小的 TLBI，遗漏其他 ASID、PE、VMID 或 Stage。
- 直接把 Table descriptor 改成 Block descriptor，造成 overlapping translation。
- 把 `ISB` 理解成替代 `DSB`；两者解决的问题不同。

## 一句话总结

> **配置 Stage 1 要同时建立 TTBR/TCR/MAIR/页表/SCTLR；修改映射则必须让页表写入、TLBI 和同步按架构顺序完成，改变 mapping size 时尤其要遵守 BBM。**

## 来源与相关资料

- [MMU.pdf](../../99_SOURCE/Training/MMU.pdf)：PDF 第 23、28–31 页，Stage 1 配置、TLBI、TLB、mapping update 与 BBM。
- [Inbox 13：ARM MMU 常见 Fault](../../01_INBOX/ChatGPT/13_ARM_MMU常见Fault及产生原因.md)：TLB Conflict Abort 与维护错误。
- [Translation Descriptors 与 Memory Attributes](../Translation/03_Translation_Descriptors与Memory_Attributes.md)。
