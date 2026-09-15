# DPT：为 Full ATS 补充 Device/VM 级物理权限检查

记录日期：2026-09-03  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，培训材料 DPT 截图（时间点未知）  
状态：Inbox；DPT 格式、walk、cache、fault、maintenance 与具体产品版本支持待结合后续 MMU S3 文档校验

## 问题

Full ATS 已经把最终 PA 交给 Device，GPC/GPT 又只能检查目标 PA 属于哪个 PAS。若两个 Realm/VM 都位于 Realm PAS，如何阻止一个 Device 使用伪造的 Translated PA 访问另一个 Realm/VM 的内存？DPT 为什么能兼顾 Full ATS 的灵活性和 Split-stage ATS 的安全性？

## 回答

### 核心结论

> **DPT（Device Permission Table）把 Stage 2 中与 Device/VM 归属相关的物理访问权限检查单独抽出来：Full ATS transaction 不必重新执行 IPA→PA translation，但 SMMU 仍可用当前 Stream 的 VMID 检查“这个 Device/VM 是否有权访问这个 PA”。**

## 1. 为什么只有 GPC 还不够

GPC/GPT 的检查粒度是 Physical Address Space（PAS）：

```text
PA → GPT → Root / Realm / Secure / Non-secure PAS
```

它能够阻止 Realm transaction 越权访问 Root、Secure 或 Non-secure PAS，但无法仅凭 PAS 区分同属 Realm PAS 的不同 Realm/VM。

例如：

```text
Realm A → VMID 10 → PA_A（Realm PAS）
Realm B → VMID 20 → PA_B（Realm PAS）
```

Realm A 的恶意 Device 伪造：

```text
SID = 5
AT = Translated
Address = PA_B
```

GPC 看到：

```text
Transaction PAS = Realm
GPT[PA_B] = Realm
       ↓
PAS 层面匹配
```

因此，GPC 可以防止“跨世界”，却不能单独防止同一 Realm PAS 内不同 Realm/VM 之间的越权访问。

## 2. DPT 增加的检查维度

培训材料把 DPT 描述为位于内存中的 lookup table，用于为 physical memory region 指定允许访问的 VMID：

```text
PA Region A → permitted VMID = 10
PA Region B → permitted VMID = 20
```

DPT 的问题不是：

> “这个 PA 应该翻译成什么？”

而是：

> **“这个 PA 允许哪个 VMID 的 Device/Stream 访问？”**

## 3. Full ATS + DPT 的检查流程

```text
Full ATS Translated transaction
SID = 5，Address = PA_B
          │
          ├──────── SID → STE → 当前 VMID = 10
          │
          └──────── PA_B → DPT → permitted VMID = 20
                                  │
                              比较 VMID
                                  │
                              10 != 20
                                  │
                                 Fault
```

抽象成公式：

```text
Transaction identity：SID → STE → VMID
Target permission：   PA  → DPT → permitted VMID
                              ↓
                        Match → Pass
                     Mismatch → Fault
```

因此 DPT 仍依赖 Stream Configuration Lookup。Full ATS 可以省掉 S1/S2 translation，但不能省掉 `SID → STE`，因为 SMMU 必须从 STE 获得当前 transaction 所属的 VMID。

## 4. DPT 与 Stage 2 的区别

Split-stage ATS 的真实 DMA 仍执行完整 Stage 2：

```text
Device ATC：IOVA → IPA
       ↓
SMMU Stage 2
       ├─ IPA → PA translation
       └─ Stage 2 permission check
```

Full ATS + DPT：

```text
Device ATC：IOVA → PA
       ↓
SMMU 不重新翻译 PA
       ↓
DPT：PA + VMID permission check
       ↓
Memory
```

所以：

> **Stage 2 同时承担 translation 与 permission；DPT 不做 IPA→PA，只保留面向最终 PA 的 Device/VM ownership check。**

## 5. 为什么说它兼顾两种 ATS 模式的优势

| 方案 | Device ATC 保存 | 后续执行 Stage 2 | Device/VM 级物理隔离 |
| --- | --- | --- | --- |
| Full ATS + GPC | IOVA → PA | 否 | GPC 只到 PAS，Realm/VM 间不充分 |
| Split-stage ATS | IOVA → IPA | 是 | 由 Stage 2 提供 |
| Full ATS + DPT | IOVA → PA | 否 | 由 DPT 的 VMID permission check 补充 |

Full ATS 的灵活性包括：

- Device 获得最终 PA。
- 更适合基于 PA 的 fully coherent Device cache。
- 更自然地支持无需经 Host Root Port 的 direct PCIe P2P routing。

DPT 在不让每笔访问重新进行 Stage 2 translation 的前提下，为这些 Full ATS transaction 增加按 VMID 的访问控制。

## 6. GPT/GPC、DPT 与 Stage 2 的分层

```text
GPT / GPC
PA → PAS
回答：“目标属于哪个安全世界？”

DPT
PA → permitted VMID
回答：“这个世界里的哪个 VM/Device Context 可以访问？”

Stage 2
IPA → PA + permissions
回答：“该 VM Context 的 IPA 如何映射到 PA，并具有什么权限？”
```

可以压缩成：

> **GPC 防跨 PAS，DPT 防同一 PAS 内跨 VM，Stage 2 则提供 IPA→PA translation 与页级权限。**

DPT 的精确权限粒度、是否只有单个 permitted VMID、表项格式和 fault 行为，应以对应版本规范为准，本文保留培训材料的机制层理解。

## 7. 产品版本边界

培训 slide 明确把 DPT 标为：

> **A future MMU S3 r1 feature**

因此不能把它表述为当前 MMU-720AE r0p2 已经具备的功能。当前项目是否采用后续版本、是否支持 DPT，以及 DPT 与 RME/GPC 的真实连接方式，都需要结合产品版本与集成资料确认。

## 面试版总结

> **DPT 用于弥补 Full ATS 的 Device/VM 级权限缺口。Full ATS transaction 已携带最终 PA，因此 SMMU不重新执行 Stage 2 translation；它通过 StreamID 查询 STE 得到当前 VMID，再以目标 PA 查询 DPT 得到允许访问该物理区域的 VMID，只有两者匹配才允许访问。GPC 只检查 PAS，无法区分同一 Realm PAS 内的不同 Realm/VM；DPT 提供更细的 VMID permission check，从而尝试兼顾 Full ATS 的 PA 可见性与 Split-stage ATS 的隔离能力。**

## 一句话总结

> **GPT 判断“这个 PA 属于哪个世界”，DPT 判断“这个 PA 允许这个世界里的哪个 VM 访问”。**

## 关联记录

- [Full ATS 的安全边界](36_Full_ATS的安全边界.md)
- [Full ATS 与 Split-stage ATS](40_Full_ATS与Split-stage_ATS.md)
- [Configuration Table Walk 与 GPT Walk](02_Configuration_Table_Walk与GPT_Walk.md)
- [PTW、HTTU、DPT 与 QTW 分流](08_PTW_HTTU_DPT与QTW分流.md)
- [Split-stage ATS 的功能限制](42_Split-stage_ATS的功能限制.md)

