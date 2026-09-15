# SMMU 与 I/O Device 的三种集成方式

记录日期：2026-09-03  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，培训材料截图（时间点未知）  
状态：Inbox；Cached/Lookaside 与 MMU-720AE TBU 类型、ATS 模式和 GPC enforcement point 的精确对应待结合培训材料及 TRM 校验

## 问题

SMMUv3 与 I/O Device 的 Inline、Cached 和 Lookaside 三种集成方式分别如何工作？它们的 translation request 与真实 DMA 数据路径有什么区别？

## 回答

### 核心结论

理解三种方式时，最关键的判别问题不是“translation 在哪里发生”，而是：

> **真正的 DMA payload 是否仍然经过 SMMU 的数据通路？**

- **Inline**：数据经过 SMMU，SMMU 在访问路径上现场翻译。
- **Cached**：Device 提前获得并缓存翻译，但数据仍经过 SMMU；已翻译请求不重复执行相同 translation。
- **Lookaside**：SMMU 只提供 translation service，Device 获得结果后直接向系统发送数据访问。

## 1. Inline Integration：边走边翻

Inline 模式下，每笔真实 DMA transaction 都进入 SMMU，并在数据访问路径上完成 translation 与 permission check：

```text
Device
  │ Untranslated address / IOVA
  ▼
SMMU datapath
  ├─ TLB hit：直接得到结果
  └─ TLB miss：执行 page-table walk
  ▼
Translated address / PA
  ▼
System Memory
```

主要特点：

- SMMU 位于真实 DMA 数据通路中。
- Device 不需要保存供后续直接使用的 translation result。
- 每个访问都由 SMMU 在数据路径上执行相应 translation/permission enforcement。

在 MMU-720AE 中，ACE-Lite TBU 的 `TBS → TBU → TBM` 数据通路可作为典型 Inline 思路理解；精确对应关系仍应以 TRM 的接口定义为准。

> **Inline：数据来了再翻，Device 自己不缓存 translation。**

## 2. Cached Integration：提前翻，但数据仍经过 SMMU

Cached 模式先通过 translation request 获得结果，并缓存在 Device 的 ATC（Address Translation Cache）中：

```text
Device
  │ Translation Request
  ▼
SMMU
  │ Translation Result
  ▼
Device ATC
```

真正 DMA 到来时：

```text
Device ATC hit
      ↓
Translated transaction
      ↓
SMMU datapath
      ↓ 不重复执行已经完成的相同 translation
System
```

因此必须区分：

```text
Translated transaction 不需要重复 translation
                         ≠
Translated transaction 绕开 SMMU
```

Cached integration 的核心是：**translation 被提前并缓存，但 SMMU 仍位于真实 DMA 路径中。**

## 3. Lookaside Integration：Translation-as-a-Service

Lookaside 模式把 translation request 路径与真实数据路径分开：

```text
Translation control path：
Device ──Translation Request──> SMMU
Device <──Translation Result─── SMMU

DMA data path：
Device ──Translated access────────────> System Memory
```

SMMU 在这里更像 Address Translation Server：它回答“地址如何翻译”，但真实 DMA payload 不需要再经过 SMMU 的数据通路。

结合 MMU-720AE 架构，可以把 LTI TBU 理解为典型 Lookaside 思路：Device 通过 Lookaside 接口请求 translation，获得响应后自行发起真正的数据访问。该对应属于结合产品架构的解释，不是培训截图本身直接给出的结论。

> **Lookaside：SMMU 只帮 Device 查地址，数据访问由 Device 直接送往系统。**

## 4. 三种方式对比

| Integration | Device 是否提前获得 translation | Device 是否需要 ATC/等效缓存 | DMA payload 是否经过 SMMU | SMMU 主要角色 |
| --- | --- | --- | --- | --- |
| Inline | 否 | 不要求 | 是 | 数据路径上的实时 translation/enforcement |
| Cached | 是 | 是 | 是 | 提供 translation，并继续位于数据路径 |
| Lookaside | 是 | 通常需要保存结果 | 否 | Translation-as-a-Service |

最简路径对比：

```text
Inline：
Device ──IOVA──> SMMU ──PA──> Memory

Cached：
Device/ATC ──Translation Request──> SMMU
Device/ATC ──Translated DMA───────> SMMU ──> Memory

Lookaside：
Device ──Translation Request──> SMMU
Device <──Translation Result─── SMMU
Device ──Translated DMA────────────────────> Memory
```

## 5. 与 ATS 的关系

Cached 和 Lookaside 都可能出现“Device 提前请求 translation 并缓存结果”的行为，因此都容易与 ATS 联系起来；但 ATS 的 translation control path 与系统集成的数据路径类型不能简单画等号。

需要分别判断：

1. Device 如何发送 translation request。
2. 返回的是完整还是阶段性的 translation result。
3. 后续 Translated DMA 是否仍进入 SMMU。
4. SMMU 是否还需要执行 Stage 2、permission 或其他检查。
5. Mapping 更新后，谁负责使 Device ATC/等效缓存失效。

因此，“支持 ATS”本身不足以推出“DMA 一定绕开 SMMU”或“一定属于 Lookaside integration”。

## 6. 三种集成方式的安全差异

### Inline 与 Cached

两者的真实 transaction 仍经过 SMMU 数据通路，因此 SMMU 可以继续执行其所在路径规定的检查。对于 Cached 模式：

> **已经 Translated 只表示不必重复同一地址翻译，并不等于可以跳过 permission、Stage 2 或 GPC 等其他适用检查。**

具体仍执行哪些检查由 translation configuration 与系统实现决定。

### Lookaside

Lookaside 中真实 DMA 不再经过 SMMU 数据通路，因此不能只因为 translation result 来自 SMMU，就断言后续访问仍由 SMMU 保护。

系统需要在其他位置保证：

- Device 只能使用合法、未失效的 translation result。
- Device/ATC 的 translation 生命周期和 invalidation 正确。
- Stage 2、地址范围、防火墙或其他访问控制没有被绕过。
- RME 系统中的 DMA 仍经过适当的 GPC enforcement point。

因此 Translation-as-a-Service 会把更多正确性与信任责任交给 Device 及系统集成。

## 7. 与 RME GPC 的关系

GPC/GPT 解决的是最终 physical granule 的 PAS 归属检查，与地址是否已经翻译是两个层次：

```text
Translation
→ 地址最终去哪里

GPC/GPT
→ 目标 physical granule 属于哪个 PAS，当前 transaction 是否允许访问
```

- Cached 路径仍经过 SMMU/相关数据通路，较容易在该路径继续执行 GPC 检查。
- Lookaside 路径直接进入 System，必须确保 SMMU datapath 之外仍有 GPC enforcement point；否则不能从“翻译由 SMMU 提供”推导出“真实 DMA 已完成 PAS 检查”。

GPC 的具体部署位置属于 MMU-720AE 与 SoC 集成事实，需要结合 DGW/AGW、互连及内存路径核实。

## 8. 缓存一致性与失效

Cached 与 Lookaside 都可能让 Device 保存 translation result。页表、权限、Device 归属或 translation context 改变后，需要正确处理：

```text
SMMU translation/configuration cache maintenance
+ Device ATC/等效缓存 invalidation
+ completion synchronization
+ outstanding DMA 的收敛
```

否则 Device 可能继续使用 stale translation。Lookaside 因数据不再经过 SMMU，尤其依赖这套生命周期管理的完整性。

## 面试版总结

> **Inline、Cached 和 Lookaside 的根本区别，是实际 DMA payload 是否经过 SMMU。Inline 模式下，所有访问经过 SMMU并现场翻译；Cached 模式允许 Device 提前把 translation 缓存在 ATC，但 Translated DMA 仍经过 SMMU，只是不重复相同 translation；Lookaside 模式中，SMMU只提供 Translation-as-a-Service，Device 获得结果后直接访问 System，DMA payload 不再经过 SMMU。因此 Lookaside 需要系统在 SMMU datapath 之外保证权限、失效和 GPC 等安全边界。**

## 一句话总结

> **Inline 是边走边翻；Cached 是提前翻、数据仍过 SMMU；Lookaside 是 SMMU 只查地址、数据直接进系统。**

## 关联记录

- [TBU、TCU、DTI 与 DTI-ATS 概述](01_TBU_TCU_DTI与DTI-ATS概述.md)
- [DGW 与 AGW 的区别](04_DGW与AGW的区别.md)
- [PCIe ATS 与 ATC 工作流程](35_PCIe_ATS与ATC工作流程.md)
- [Full ATS 的安全边界](36_Full_ATS的安全边界.md)

