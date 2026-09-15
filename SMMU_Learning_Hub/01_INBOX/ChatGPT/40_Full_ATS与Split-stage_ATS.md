# Full ATS 与 Split-stage ATS：地址语义、STE.EATS 与后续处理

记录日期：2026-09-03  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，培训材料截图及连续追问（截图时间点未知）  
状态：Inbox；`STE.EATS` 编码、AXI ATS flow sideband、output attributes 和 security checks 的精确范围待结合 SMMUv3、AXI 与 MMU-720AE 文档校验

## 问题

PCIe ATS 的 Full ATS 与 Split-stage ATS 有什么区别？PCIe Endpoint 是否知道 ATC 中保存的是完整翻译还是 Stage 1 翻译？SMMU 收到 Translated transaction 后，如何知道输入地址应按 PA 还是 IPA 解释？Full ATS 已经完成地址翻译，为什么仍需要查询 STE，是否还会发生 table walk？

## 回答

### 核心结论

> **Full ATS 把 Stage 1 + Stage 2 的完整结果缓存到 Device ATC，后续地址按 PA 处理；Split-stage ATS 只缓存 Stage 1 的 IOVA→IPA，真实 DMA 到达 SMMU 后仍执行 Stage 2。`Translated` 只说明地址已经经过 ATS，`STE.EATS` 才说明它究竟已经翻译到哪一级。**

## 1. 两种模式的基本路径

以下讨论建立在培训材料的 Cached Integration 场景上：Device 可以提前请求 translation，真实 DMA 仍经过 SMMU/TBU 数据通路。

### Full ATS

ATS Translation Request 完成所有已启用的 translation stage：

```text
IOVA
  ↓ Stage 1
IPA
  ↓ Stage 2
PA
  ↓
ATS Completion 返回完整结果
  ↓
Device ATC：IOVA → PA
```

ATC 命中后的真实 DMA：

```text
Device
  │ Address = PA
  │ AT = Translated
  ▼
SMMU / TBU
  │ 不再重复执行 Stage 1 / Stage 2 translation
  │ 仍执行该路径规定的配置、安全和属性检查
  ▼
System
```

### Split-stage ATS

ATS 只提前完成 Stage 1：

```text
IOVA
  ↓ Stage 1
IPA
  ↓
ATS Completion 返回 Stage 1 output
  ↓
Device ATC：IOVA → IPA
```

ATC 命中后的真实 DMA：

```text
Device
  │ Address = IPA
  │ AT = Translated
  ▼
SMMU / TBU
  │ Stage 2：IPA → PA
  │ Stage 2 permission check
  ▼
System
```

## 2. 模式对比

| 项目 | Full ATS | Split-stage ATS |
| --- | --- | --- |
| ATS 提前完成的阶段 | Stage 1 + Stage 2 | Stage 1 |
| ATC 中的结果 | IOVA → PA | IOVA → IPA |
| Translated DMA 输入 SMMU 时的地址语义 | PA | IPA |
| 后续是否再做 Stage 1 | 否 | 否 |
| 后续是否再做 Stage 2 | 否 | 是 |
| 最终 PA 由谁在真实 DMA 时决定 | Device 使用 ATC 中的完整结果 | SMMU/Hypervisor 控制的 Stage 2 |
| 对不可信 Device 的隔离 | 更依赖 Device 信任和其他安全机制 | 保留 Stage 2 作为 VM/Host 边界 |

最简记忆：

> **Full ATS：Device 拿到 PA；Split-stage ATS：Device 只拿到 IPA，PA 仍由 SMMU 决定。**

## 3. 追问一：PCIe Endpoint 是否知道 Full 或 Split-stage？

PCIe Endpoint 不需要根据每笔 transaction 自行判断“这个地址是 IPA 还是最终 PA”。从 Device 的工作视角看：

```text
原始地址
  ↓ ATC lookup
ATS Completion 返回的 Translated Address + permissions
  ↓
ATC 缓存结果
  ↓
Device 发送 AT = Translated 的 transaction
```

无论 SMMU 使用哪种模式，Device 发出的形式都可以是：

```text
AT = Translated
Address = ATC 中缓存的 Translated Address
```

PCIe transaction 不需要再携带一个单独的 `Full_ATS` 或 `Split_ATS` 位。更准确地说：

> **Full/Split-stage 是 SMMU 对该 Stream 的 translation 策略，不是由每个 PCIe TLP 中一个额外的模式字段来决定。**

Device 或其软件栈可能知道平台对该 Stream 的总体配置，但在数据路径上，Endpoint 只需正确使用 ATS Completion 返回的地址和权限；系统内部把它解释为 IPA 还是 PA，由 SMMU Stream Configuration 决定。

## 4. 追问二：SMMU 如何知道地址是 IPA 还是 PA？

SMMU 不能根据地址数值本身判断。例如 `0x8000_4000` 是 IPA 还是 PA，取决于 translation regime 中的语义，而不是 bit pattern。

判断需要两类信息：

```text
Transaction sideband / flow
→ 表明该 transaction 已经过 ATS（Translated）

StreamID → STE.EATS
→ 表明该 Stream 使用 Full ATS 还是 Split-stage ATS
```

概念决策流程：

```text
ATS Translated transaction
Address + StreamID
        ↓
SID → STE
        ↓
读取 STE.EATS
   ┌────┴──────────────┐
   ▼                   ▼
Full ATS          Split-stage ATS
Address 按 PA     Address 按 IPA
不做 S1/S2        继续做 Stage 2
```

在当前问答所依据的 SMMUv3 配置中，可记为：

```text
STE.EATS = 0b01 → Full ATS
STE.EATS = 0b10 → Split-stage ATS
```

精确编码和保留值仍应按当前采用的 SMMUv3 revision 查证。

最关键的一句是：

> **`Translated` 告诉 SMMU“已经经过 ATS”；`STE.EATS` 告诉 SMMU“究竟已经翻到了哪一级”。**

## 5. Full ATS 的完整示例

假设：

```text
IOVA 0x1000
  ↓ Stage 1
IPA  0x4000
  ↓ Stage 2
PA   0x8000_4000
```

ATS Request 阶段：

```text
Device → Root Port → DTI-ATS → TCU
                           ↓ S1 + S2
Device ← ATS Completion：PA 0x8000_4000
```

ATC 保存：

```text
IOVA 0x1000 → PA 0x8000_4000
```

真实 DMA：

```text
Address = 0x8000_4000
AT = Translated
SID = X
      ↓
SMMU：SID → STE → EATS = Full
      ↓
地址按 PA 处理，不再执行 S1/S2
      ↓
继续该路径要求的 security/attribute processing
```

## 6. Split-stage ATS 的完整示例

ATS Request 只完成 Stage 1：

```text
Device → Root Port → DTI-ATS → TCU
                           ↓ S1
Device ← ATS Completion：IPA 0x4000
```

ATC 保存：

```text
IOVA 0x1000 → IPA 0x4000
```

真实 DMA：

```text
Address = 0x4000
AT = Translated
SID = X
      ↓
SMMU：SID → STE → EATS = Split-stage
      ↓
地址按 IPA 处理
      ↓ Stage 2
PA = 0x8000_4000
```

Stage 2 的 page table、VMID 和 permission 仍由 Hypervisor/SMMU 控制，因此 Device 即使伪造 IPA，也不能自然获得任意 Host PA 的访问权限。

## 7. 追问三：Full ATS 为什么仍要查询 STE？

Full ATS 省掉的是 **address translation**，不是 **Stream configuration identification**。

SMMU 收到的 transaction 只有类似信息：

```text
StreamID = X
ATS status = Translated
Address = ...
```

它仍需通过 SID 找到 STE，确认：

- 该 Stream 是否允许 ATS/Translated traffic。
- `STE.EATS` 是 Full 还是 Split-stage。
- 输入地址应按 PA 还是 IPA 解释。
- 还需执行哪些适用的安全、属性或 Stage 2 处理。

因此 Full ATS 的典型配置查找是：

```text
SID
  ↓
Configuration Cache
  ├─ Hit：直接得到缓存的 STE
  └─ Miss：从内存中的 Stream Table fetch STE
           ↓
        填充 Configuration Cache
```

“仍要查 STE”不等于每笔 transaction 都去 DDR 读取 STE。正常情况下可命中 Configuration Cache。

## 8. Configuration Lookup 与 Translation Walk 必须区分

```text
Configuration Lookup
SID → STE；必要时 SSID → CD
回答：“这笔 transaction 应该怎么处理？”
```

```text
Translation Lookup / Walk
IOVA 或 IPA → TLB → Translation Table Walk → output address
回答：“这个地址最终翻到哪里？”
```

对于 Full ATS Translated transaction：

| 操作 | 是否需要 |
| --- | --- |
| SID → STE / Configuration Cache lookup | 需要 |
| Configuration Cache miss 后 fetch STE | 可能需要 |
| Stage 1 TLB lookup / page-table walk | 不需要 |
| Stage 2 TLB lookup / page-table walk | 不需要 |

因此术语上最好说：

> **Full ATS 可能发生 STE configuration-table fetch，但不会因此重新执行 S1/S2 translation-table walk。**

Split-stage ATS 则不同：完成 STE lookup 后，还要进行 Stage 2 TLB lookup；若 miss，可能发生 Stage 2 page-table walk。

## 9. DTI-ATS 与 DTI-TBU 的分工

两种 ATS 模式还可以帮助区分两个请求源：

```text
DTI-ATS
PCIe Controller ↔ TCU
→ Device 提前发 ATS Translation Request / 接收 Response
```

```text
DTI-TBU
ACE-Lite TBU ↔ TCU
→ TBU 处理真实 transaction 时，为所需 translation 向 TCU 请求服务
```

在 Split-stage ATS 的真实 DMA 中，如果 TBU 的 Stage 2 translation cache miss，就可能通过 DTI-TBU 向 TCU 请求 Stage 2 translation；Full ATS 则不需要重新请求 S1/S2 translation，但仍需识别 Stream Configuration。

## 10. 安全边界

Full ATS 中，SMMU 根据 `STE.EATS` 知道输入地址应按 PA 处理，但不会通过重新执行 S1+S2 来证明“这个 PA 一定是之前 ATS Completion 合法返回的那个地址”；否则会抵消 Full ATS 的性能价值。

因此：

- Full ATS 更依赖 Endpoint 正确使用 ATC 结果及系统对 Device 的信任。
- Split-stage ATS 保留 SMMU Stage 2，可继续执行 VM/Host page-level isolation。
- Full ATS 仍可接受 GPC、DPT 或其他适用安全检查，但这些机制解决的粒度不同。
- GPC 可限制跨 PAS 访问，却不等价于验证某个 PA 是否属于该 Device 的 ATS 授权。
- DPT 的目标可理解为针对 translated access 增加按 Device/Stream 的更细粒度物理访问约束；精确机制需要结合 DPT 专题和规范确认。

详细威胁模型与 GPC 边界见关联记录 36，本记录只保留解释两种 ATS 模式所必需的安全结论。

## 面试版总结

> **Full ATS 会把 Stage 1 + Stage 2 的完整 translation 缓存在 Device ATC 中，后续 Translated DMA 的地址按 PA 处理，不再执行 S1/S2；Split-stage ATS 只缓存 Stage 1 的 IOVA→IPA，真实 DMA 到达 SMMU 后仍执行 Stage 2。PCIe transaction 只表明地址已经经过 ATS，并不单独携带 Full/Split 模式；SMMU通过 StreamID 查 STE，并根据 `STE.EATS` 决定该地址是 PA 还是 IPA。即使是 Full ATS，SMMU仍需做 Configuration Lookup，但这通常命中 Configuration Cache，只有 miss 时才 fetch STE，并不等于重新进行 translation-table walk。**

## 一句话总结

> **`Translated` 说明“已经翻过”，`STE.EATS` 说明“翻到哪一级”：Full ATS 到 PA，Split-stage ATS 到 IPA并保留 Stage 2。**

## 关联记录

- [Configuration Lookup 与 Translation Lookup](29_Configuration_Lookup与Translation_Lookup.md)
- [PCIe ATS 与 ATC 工作流程](35_PCIe_ATS与ATC工作流程.md)
- [Full ATS 的安全边界](36_Full_ATS的安全边界.md)
- [SMMU 与 I/O Device 的三种集成方式](39_SMMU三种设备集成方式.md)

