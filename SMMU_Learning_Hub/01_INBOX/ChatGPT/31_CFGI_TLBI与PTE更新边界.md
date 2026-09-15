# CFGI、TLBI 与 PTE 更新边界

> 记录日期：2026-09-01  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 状态：Inbox 问答整理，具体字段更新所需的 Invalidation 类型与顺序待结合 SMMUv3 Architecture Specification 校验

## 凝练问题

**修改 STE/CD 中的 ASID、TTB0 等 Translation Configuration 后，执行 CFGI 是否会同步修改或修正相关 PTE？CFGI、TLBI 与软件更新页表分别负责什么？**

## 核心结论

> **不会。CFGI 只使 SMMU 内部缓存的 STE/CD Configuration 失效，不会修改内存中的 PTE；TLBI 使旧的 Cached Translation Result 失效，也不会修改 PTE。STE、CD 和 PTE 的实际内容都必须由软件更新，再按照修改对象和架构规则执行相应 CFGI、TLBI 与 SYNC。**

一句话记忆：

> **软件改配置和页表；CFGI 清配置缓存；TLBI 清翻译缓存；SYNC 确认维护完成。**

## 1. 必须分清三层状态

```text
Configuration Memory
STE / CD
├─ ASID
├─ TTB0
├─ VMID
├─ S2TTB
└─ Translation Settings
        ↓
SMMU Configuration Cache

Translation Table Memory
PTE / Descriptor
├─ Output Address
├─ Permission
├─ AF
├─ AttrIndx
└─ 其他 Mapping Attributes
        ↓
Page-table Walk

Cached Translation Result
VA/IOVA/IPA → Output Address + Permissions
        ↓
TLB / Translation Cache
```

三者解决的问题不同：

| 状态 | 回答的问题 | 典型缓存维护 |
|---|---|---|
| STE/CD | 使用哪种 Translation Context、页表在哪里、如何解释 | CFGI |
| PTE/Translation Table | 某个地址具体映射到哪里、权限和属性是什么 | 软件修改内存内容 |
| TLB/Cached Translation | 先前计算出的 Translation Result | TLBI |

## 2. CFGI 不会修改 PTE

假设软件把 CD 从：

```text
ASID = 10
TTB0 = PageTable_A
```

改成：

```text
ASID = 20
TTB0 = PageTable_A
```

这里改变的是 CD 中的 Stage 1 Context 标识，`PageTable_A` 中的 PTE 没有发生变化。

概念流程是：

```text
软件修改 Memory 中的 CD
          ↓
执行相应 CMD_CFGI
          ↓
SMMU 丢弃旧的 CD/Configuration Cache
          ↓
后续重新读取 CD
          ↓
观察到新的 ASID = 20
```

因此：

> **CFGI 让新的 Configuration 被 SMMU 重新观察和使用，不负责修改该 Configuration 指向的页表。**

## 3. 修改 TTB0 时也是同一边界

例如：

```text
旧 CD.TTB0 → PageTable_A
新 CD.TTB0 → PageTable_B
```

执行相应 CFGI 后，SMMU 的旧 CD Cache 被清除，后续 Configuration Lookup 才能获取新的 `TTB0`。

但：

```text
PageTable_A 中的 PTE
PageTable_B 中的 PTE
```

都不会因为 CFGI 自动发生变化。

> **CFGI 改变的是 SMMU 以后“去哪里找页表、按什么 Context 解释”；Page-table Memory 仍由软件显式管理。**

## 4. 真正修改 PTE 时需要什么

假设软件把映射从：

```text
IOVA 0x4000 → PA_A
```

改成：

```text
IOVA 0x4000 → PA_B
```

软件需要实际写入相应 PTE。由于旧 Translation 可能仍缓存在 TLB 中，概念流程通常包含：

```text
软件修改 PTE
   ↓
保证 PTE 更新按要求可见和有序
   ↓
执行相应 TLBI
   ↓
使旧 Cached Translation 失效
   ↓
执行/等待相应同步完成
```

这里：

- 修改 PTE 的动作由软件完成。
- TLBI 只清除旧 Translation Result，不会替软件写入新 PTE。
- 后续 TLB Miss 时，SMMU 才会从更新后的页表重新 Walk 并生成新 Translation。

## 5. 为什么某些 Context 修改同时需要 CFGI 与 TLBI

以修改 `CD.TTB0` 为例，SMMU 可能同时保留：

```text
Configuration Cache
→ 旧 CD，仍指向 PageTable_A

Translation Cache/TLB
→ 旧 VA → PA_A Translation Result
```

只做 CFGI：

```text
新的 CD 可以被重新读取
但旧 TLB Entry 仍可能命中
```

只做 TLBI：

```text
旧 Translation Result 被清除
但旧 CD Cache 可能仍让后续 Walk 使用 PageTable_A
```

因此某些会改变 Translation Context 或使既有 Translation 不再正确的 STE/CD 更新，需要组合：

```text
修改 STE/CD
   ↓
CFGI：清除旧 Configuration Cache
   ↓
相应 TLBI：清除受影响的 Cached Translation
   ↓
CMD_SYNC/架构规定的同步：确认完成
```

但不能机械地规定“所有 STE/CD 修改都固定执行 CFGI + TLBI + SYNC”。具体需要哪种 Invalidation、作用范围和顺序，必须按被修改字段的架构更新规则执行。

## 6. ASID 为什么不是 PTE 修改

ASID 属于 Stage 1 Translation Context：

```text
SMMU Stage 1：CD.ASID
CPU Stage 1：TTBR 中的 ASID Field/当前 Context
```

PTE 则描述某个地址的具体映射与属性：

```text
Output Address
AP/Permission
AF
AttrIndx
SH
Execute-never 等
```

所以修改 CD.ASID 并不意味着要把整棵页表的每个 PTE 一起修改。ASID 作为 Context/Cache Tag 区分地址空间，而页表本身由 TTB 指向。

## 7. 按修改对象选择维护方式

```text
修改 STE/CD
→ 修改“用什么 Context、怎么翻、页表在哪里”
→ 相应 CFGI

修改 PTE
→ 修改“具体翻到哪里、具有什么权限/属性”
→ 软件写 PTE + 相应 TLBI

修改 STE/CD 且旧 Translation 也不再有效
→ 按规范执行 CFGI + 相应 TLBI + Synchronization
```

## 面试式回答

> **CFGI 是 Configuration Cache Invalidation，不是 Page-table Update。软件修改 STE/CD 后，通过 CFGI 让 SMMU 丢弃旧的 Configuration Cache；它不会修改任何 PTE。ASID 本身也不在 PTE 中，而属于 Translation Context。如果页表映射发生变化，软件必须单独更新 PTE，并用相应 TLBI 使旧 Cached Translation 失效。若 STE/CD 的变化同时使已有 Translation 无效，则还要按照 SMMUv3 对具体字段的更新规则组合 CFGI、TLBI 和同步操作。**

## 一句话总结

> **CFGI 清“怎么翻”的旧缓存，TLBI 清“翻译结果”的旧缓存，而 PTE/STE/CD 的内容始终由软件自己修改。**

## 相关记录与资料

- [SMMUv3 为什么使用 Command Queue 进行维护](24_SMMUv3为何使用Command_Queue维护.md)
- [CMDQ、PROD/CONS 与 CMD_SYNC](30_CMDQ_PROD_CONS与CMD_SYNC.md)
- [Configuration Lookup 与 Translation Lookup](29_Configuration_Lookup与Translation_Lookup.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)

