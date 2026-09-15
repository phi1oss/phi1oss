# ARM MMU Fault 与定位

> 整理日期：2026-08-30  
> 状态：正式知识文档，待 Fault Case 练习验证

## 核心结论

**MMU fault 是 translation、descriptor 状态或权限检查无法产生合法访问结果时触发的同步异常。定位时不要只看 fault 名称，而要恢复当时的 translation regime，按真实 table-walk 路径检查地址范围、descriptor、AF、权限和 TLB maintenance。**

## 1. 五类核心 Fault

### Address Size Fault

输入 VA 或 descriptor 中的 output address 超出当前 regime/实现支持的地址宽度。

> 地址数值本身不能被当前配置合法表示。

产生 Address Size Fault 的 translation entry 不会被缓存，因此修复地址或配置后通常不需要为该 fault entry 做 TLBI；但若同时修改其他已缓存 mapping，仍需按对应规则维护。

### Translation Fault

Walk 在某一级遇到 Invalid/Fault descriptor，或配置明确禁止相应 walk，无法找到有效 leaf mapping。

> 页表中没有通向输出地址的有效路径。

Translation Fault 不等于操作系统层所有“Page Fault”；后者是更高层软件概念。

### Access Flag Fault

Descriptor 有效且权限可能正确，但 `AF=0`，并且硬件 AF 自动更新未启用或不可用。

> Mapping 存在，但尚未标记为已经访问。

软件可以在异常处理程序中更新 AF，并执行所需维护后重试；启用 HA/HTTU 类能力时，硬件可在规定条件下原子更新 AF。

### Permission Fault

Mapping 存在，但当前 Read/Write/Execute、Exception Level 或 Stage 权限不允许访问，例如写只读页、EL0 访问 privileged-only 区域或执行 XN 页面。

> 有路，但当前访问方式没有权限走。

若软件修改页表权限以处理 fault，必须使可能缓存旧权限的 TLB entry 失效。

### TLB Conflict Abort

同一 lookup 条件同时匹配多个彼此冲突的 translation result，例如旧 4KB mapping 与新 2MB mapping 同时覆盖同一 VA。

> 找到多条路，但结果互相冲突。

典型根因是错误的页表更新、遗漏 TLBI 或违反 BBM。是否报告该 abort 可以是 IMPLEMENTATION DEFINED，但出现冲突 translation 本身已经说明软件维护流程错误。

## 2. Fault 在 Translation Flow 中的位置

```text
PE issues VA
    ↓
Address range / size check
    └─ illegal → Address Size Fault
    ↓
TLB Lookup
    ├─ conflicting hits → TLB Conflict Abort
    └─ Miss
         ↓
      Table Walk
         ↓
Descriptor valid/type check
    └─ invalid/no mapping → Translation Fault
    ↓
Access Flag check
    └─ AF=0 and no hardware update → Access Flag Fault
    ↓
Permission / execute / stage checks
    └─ denied → Permission Fault
    ↓
PA + Attributes
    ↓
Physical Memory Access
```

实际架构可能在不同阶段提前或合并检查；该图用于建立定位顺序，不代替异常优先级的精确定义。

## 3. Training 中列出的其他根因

- Address authentication failure。
- `TCR_ELx.TnSZ` 小于实现允许的最小值。
- Translation table descriptor 使用无效编码。
- 使用实现不支持的地址扩展或 descriptor 形式。
- `EPDn` 等配置禁止对相应 TTBR 区域进行 walk。
- 误用 Contiguous Bit，group 对齐、数量、连续性或 attributes 不满足要求。
- 页表更新后遗漏必要的 invalidation 和 synchronization。

这些根因最终会表现为 Address Size、Translation、Permission、Access Flag 或其他 Data/Instruction Abort syndrome，需要结合 ESR 解码。

## 4. 定位步骤

### Step 1：确认异常上下文

- 读取 `ESR_ELx`：异常类别、fault status、读写方向和 fault level。
- 读取 `FAR_ELx`：相关 fault address。
- 确认异常来自 instruction fetch 还是 data access。

### Step 2：恢复 Translation Regime

- 哪个 EL 和 Security state？
- Stage 1 还是 Stage 2？
- 使用 TTBR0 还是 TTBR1？
- TCR 中的 VA size、granule、PA size 和 EPD 配置是什么？
- 当前 ASID/VMID 是什么？

### Step 3：手工 Walk

按 VA bitfield 逐级计算 descriptor 地址，检查：

- Descriptor 是否 Valid。
- 类型和 level 是否允许。
- Next-level/output address 是否在支持范围内。
- AF、AP、UXN/PXN、NS、AttrIndx、SH 等是否符合访问。

### Step 4：检查 TLB 与更新历史

- 页表是否刚被修改？
- 修改前后是否使用正确 DSB/TLBI/DSB/ISB 序列？
- TLBI 范围是否覆盖正确 VA、ASID、VMID、Stage 和 shareability domain？
- 是否改变 mapping size 而没有 BBM？
- 是否只修改 contiguous group 的单个 descriptor？

### Step 5：修复并最小范围验证

修复配置或 descriptor 后执行架构要求的维护，再用原 fault address 重试，并验证相邻地址、其他 PE 和其他 address-space context。

## 快速对比

```text
Address Size  → 地址超出当前规则
Translation   → 没有有效 mapping
Access Flag   → mapping 有效但 AF 状态不满足
Permission    → mapping 有效但访问方式被拒绝
TLB Conflict  → 同时存在互相冲突的 cached translation
```

## 一句话总结

> **定位 MMU fault 的主线是：先用 ESR/FAR 确定类型和地址，再恢复 regime、手工 walk、检查 attributes，最后核对页表更新与 TLBI/BBM。**

## 来源与相关资料

- [MMU.pdf](../../99_SOURCE/Training/MMU.pdf)：PDF 第 24–31 页，MMU faults、TLBI、TLB conflict 与 BBM。
- [Inbox 13：ARM MMU 常见 Fault](../../01_INBOX/ChatGPT/13_ARM_MMU常见Fault及产生原因.md)。
- [Stage 1 配置与 TLB 维护](../Software/01_Stage1配置与TLB维护.md)。
