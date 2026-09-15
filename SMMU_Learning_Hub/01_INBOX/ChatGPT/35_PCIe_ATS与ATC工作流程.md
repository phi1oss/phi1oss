# PCIe ATS 与 ATC 工作流程

记录日期：2026-09-02  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，培训材料 ATS 流程截图  
状态：Inbox；Full/Split-stage ATS 的精确架构术语、响应属性及 ATC invalidation 顺序待结合 PCIe ATS、SMMUv3 和 MMU-720AE 文档校验

## 问题

PCIe ATS 的 translation request 如何经过 Root Complex 和 SMMU？Device 为什么需要 ATC？获得翻译结果后的 DMA 如何发送？页表映射变化后，如何避免 Device 继续使用旧翻译？

## 回答

### 核心结论

> **ATS 允许 PCIe Device 在真正 DMA 前主动查询地址翻译；SMMU 依据现有 translation context 计算结果，Device 将结果缓存在 ATC，后续 DMA 使用缓存结果并携带 Translated 属性，从而减少重复翻译延迟。**

## 1. ATS 解决什么问题

不使用 ATS 时，Device 通常把 untranslated address 随 DMA transaction 送入系统，由 SMMU 在数据访问路径上完成 translation：

```text
PCIe Device
      ↓ DMA：Untranslated address
Root Complex / SMMU
      ↓ Translation
Physical Memory
```

对于 GPU、NVMe、网络或 AI 加速器等高性能设备，ATS 允许 Device 提前询问将来需要使用的地址翻译，并在本地缓存答案。

## 2. ATS Translation Request 的控制路径

概念流程如下：

```text
PCIe Device
      │ ATS Translation Request
      ▼
Root Complex / Root Port
      │ DTI-ATS Translation Request
      ▼
SMMU / TCU
      │ Configuration lookup + Translation lookup/walk
      ▼
DTI-ATS Response
      │
Root Complex
      │ PCIe ATS Translation Completion
      ▼
PCIe Device
```

ATS Request 属于 translation/control traffic，不是实际承载 DMA payload 的数据 transaction。MMU-720AE 中，DTI-ATS 为 Root Port/Root Complex 与 TCU 之间提供 ATS translation service 通路。

## 3. SMMU 如何计算 ATS 结果

SMMU 不为 ATS 维护另一套独立页表。它仍然依据当前 Stream 的 translation context 工作，例如：

```text
StreamID
+ PASID / SSID（若使用）
      ↓
STE / CD configuration lookup
      ↓
Translation cache lookup
      ├─ Hit：返回缓存结果
      └─ Miss：按配置执行 Stage 1 / Stage 2 table walk
```

成功时返回 translation result 和相关属性；失败时返回相应 failure。精确响应字段及权限语义应以 PCIe ATS 与 SMMUv3 规范为准。

## 4. ATC 是 Device 侧的翻译缓存

Device 收到成功的 ATS Completion 后，将结果放入 Address Translation Cache（ATC）：

```text
Device address
      ↓ ATS
Translation result
      ↓
Device ATC
```

可以粗略类比：

| 位置 | 翻译缓存 |
| --- | --- |
| CPU | TLB |
| SMMU | TBU/TCU translation cache |
| ATS-capable Device | ATC |

ATC 命中后，Device 不需要为每笔 DMA 重新发送 ATS Translation Request。

## 5. 后续 Translated DMA

Device 使用 ATC 中的结果发起真实 DMA，并在 transaction 中表明地址已经过 ATS translation：

```text
Device ATC lookup
      ↓ Hit
Traffic tagged as Translated
      ↓
Root Complex / SMMU / System
      ↓
Memory access
```

但 `Translated` 不能简单等同于“此后完全绕开 SMMU”。后续还需执行哪些处理，取决于 Stream 配置、ATS 模式以及 Stage 1/Stage 2 的划分；在 split-stage 场景中，真实 DMA 仍可能由 SMMU 执行后续 Stage 2 translation 与 permission check。

## 6. Mapping 改变后必须处理 ATC 旧条目

例如 Device ATC 已缓存：

```text
IOVA 0x4000 → old translation
```

软件随后修改了页表。如果只维护 SMMU 内部 TLB，而没有让 Device 丢弃 ATC 旧条目，Device 仍可能继续发送基于旧 translation 的 DMA。

因此更新闭环需要同时考虑：

```text
更新页表/translation context
        ↓
执行 SMMU 所需的 CFGI / TLBI 与同步
        ↓
通过 ATS invalidation 协议使 Device ATC 旧条目失效
        ↓
确认所需维护完成
        ↓
允许依赖新映射的 DMA 继续
```

各类更新的精确顺序、完成条件与 invalidation completion 语义必须依据 PCIe ATS 和 SMMUv3 规范，不能只凭概念流程推断。

## 7. 三个概念的分工

- **ATS**：Device 如何向系统请求 translation。
- **ATC**：Device 把 translation result 缓存在哪里。
- **SMMU**：依据受软件控制的 translation context 和页表计算、管理 translation 的系统组件。

## 面试版总结

> **PCIe ATS 允许 Device 通过 Root Complex 向 SMMU预取地址翻译。MMU-720AE 中，Root Port 可通过 DTI-ATS 把 Translation Request 送到 TCU；SMMU依据 StreamID、PASID/SSID、STE/CD 和页表得到结果，再经 Root Complex 返回 Device。Device 将结果缓存在 ATC，后续 DMA 可以携带 Translated 属性，以减少重复翻译时延。映射改变时必须同步维护 SMMU translation cache 与 Device ATC，避免 stale translation。**

## 一句话总结

> **ATS 是 Device 提前问“地址怎么翻”，ATC 是 Device 保存答案，而真正的 translation context 仍由系统软件和 SMMU 控制。**

## 关联记录

- [TBU、TCU、DTI 与 DTI-ATS 概述](01_TBU_TCU_DTI与DTI-ATS概述.md)
- [Substream 如何关联进程地址空间](28_Substream如何关联进程地址空间.md)
- [Configuration Lookup 与 Translation Lookup](29_Configuration_Lookup与Translation_Lookup.md)
- [CFGI、TLBI 与 PTE 更新边界](31_CFGI_TLBI与PTE更新边界.md)
- [Full ATS 的安全边界](36_Full_ATS的安全边界.md)

