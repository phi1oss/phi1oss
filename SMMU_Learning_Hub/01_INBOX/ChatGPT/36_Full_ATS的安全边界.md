# Full ATS 的安全边界：Translated Traffic、Stage 2 与 GPC

记录日期：2026-09-02  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，对 ATS 流程的连续追问  
状态：Inbox；Full/Split-stage 精确配置、恶意 Device 威胁模型及 MMU-720AE GPC enforcement path 待结合规范和项目集成校验

## 问题

恶意 Device 能否伪造 `Traffic tagged as Translated`，直接访问任意物理地址？Full ATS 为什么需要更高的 Device 信任？Split-stage ATS、Stage 2 以及 RME 的 GPC/GPT 分别能够解决哪些风险，又有哪些边界？

## 回答

### 核心结论

> **Translated tag 只表达协议状态——“该地址声称已经翻译”，它不是地址合法性的不可伪造证明。Full ATS 把更多责任交给 Device/ATC，因此必须依靠 ATS 授权、Device 信任和系统隔离控制风险；Split-stage ATS 可保留 SMMU Stage 2 作为 VM/Host 隔离边界，RME GPC 则进一步阻止跨 PAS 的物理访问。**

## 1. 正常 ATS 与恶意 Device 的区别

正常 ATS Device 的行为是：

```text
ATS Translation Request
        ↓
SMMU 返回合法 translation result
        ↓
Device 缓存到 ATC
        ↓
仅使用该结果发送 Translated DMA
```

但在威胁模型中，恶意 Device 可能违反协议，直接构造某个地址并把 transaction 标为 `Translated`。因此：

> **Translated 属性不能单独充当 authorization token；ATS 是性能与协议机制，不是验证 Device 是否诚实的安全认证协议。**

## 2. ATS 必须经过系统授权

Device 宣称支持 ATS，并不意味着它可以自行决定使用 Translated traffic。合法启用通常需要多层配置共同成立：

```text
Device ATS capability
+ PCIe/Root Complex enable
+ SMMU Stream/translation context configuration
+ 正确的 ATS request、completion 与 invalidation flow
```

系统不应把未获授权的 Device 所发送的 Translated traffic 当作正常的 ATS DMA。具体检查点和失败行为需要结合 PCIe Root Complex 与 SMMU 集成确认。

## 3. Full ATS 的信任边界

Full ATS 可以概念化为：

```text
IOVA
  ↓ ATS Request
SMMU 完成所需 translation
  ↓
最终 translation result 进入 Device ATC
  ↓
后续 Translated request 不再重复完整 translation
```

如果 Device 获得并使用的是最终地址，而下游又不再执行等价的细粒度隔离检查，那么系统必须更强地信任 Device 会遵守 ATS 协议、只使用合法获得且未失效的结果。

因此 Full ATS 的安全原则是：

- 不对不可信 Device 无条件开放。
- 严格配置 ATS capability、StreamID/PASID 和 translation context。
- 在映射、设备归属或上下文改变时正确完成 ATC invalidation。
- 在设备直通、复位和重新分配生命周期中处理残留 translation。

## 4. Split-stage ATS 为什么能保留 Stage 2

在虚拟化场景中，可以把两个阶段的职责分开理解：

```text
Device / ATC
    Stage 1：IOVA → IPA
          ↓ Translated DMA 携带 IPA
SMMU
    Stage 2：IPA → PA + permission check
          ↓
Physical Memory
```

即使恶意 Device 伪造某个 IPA，它仍必须经过由 Hypervisor 控制的 Stage 2：

- 有合法 Stage 2 mapping：只能到达分配给该 VM/Device context 的 PA。
- 没有合法 mapping 或权限不足：产生 fault。

因此：

> **Stage 1 可以为了性能部分下沉到 Device，但 Stage 2 仍由 SMMU/Hypervisor 掌握最终 Host PA 的访问边界。**

## 5. RME GPC/GPT 为 Full ATS 增加什么保护

RME 的 Granule Protection Table 不负责 `IOVA → PA`，而是记录 physical granule 属于哪个 Physical Address Space（PAS）。如果系统集成确保 Translated traffic 仍经过 GPC enforcement path，则可以检查：

```text
Translated transaction
    PA + transaction PAS
          ↓
GPC 查询 GPT
          ↓
比较 transaction PAS 与目标 granule PAS
          ├─ 允许：继续访问
          └─ 不允许：阻止/报错
```

例如 Realm Device 伪造最终 PA 去访问 Root PAS 的 granule，GPC 可以发现 PAS 不匹配并阻止访问。因此：

> **ATS 可以减少重复地址翻译，但不应因此绕过 RME 的 physical granule protection。**

结合 MMU-720AE 的概念模型，Client-originated traffic 与 SMMU 自身发起的访问分别需要经过相应 GPC wrapper/enforcement path；DGW、AGW 的精确覆盖范围仍须用 TRM 和 SoC 连接图确认。

## 6. GPC 不能证明 ATS 结果是否合法

GPC 的检查粒度是 PAS 归属，而不是“这个 Device 是否曾合法获得这个 PA”。

假设 `PA_A` 与 `PA_B` 都属于 Realm PAS，Device 只被 ATS 合法返回过 `PA_A`，却伪造访问 `PA_B`：

```text
Transaction PAS = Realm
GPT[PA_B] = Realm
        ↓
PAS 检查可能通过
```

GPC 无法单独判断 `PA_B` 是否属于该 Device 的 per-page 授权。因此它是额外安全边界，而不是 ATS translation authentication。

## 7. 三种机制解决不同问题

| 机制 | 主要回答的问题 | 保护粒度与边界 |
| --- | --- | --- |
| ATS enable / Device trust | 是否允许该 Device 使用 ATS/Full ATS | 协议能力与信任决策，不能替代地址权限检查 |
| Stage 2 | 该 VM/Device context 能访问哪些 PA | VM/context/page 级翻译与权限边界 |
| GPC/GPT | 该 PA granule 属于哪个 PAS | Root/Realm/Secure/NS 等 PAS 边界，不能认证某次 ATS 返回结果 |

完整安全模型应理解为多层组合，而不是只依赖 Translated tag：

```text
Device trust + ATS authorization
            ↓
ATS/ATC translation lifecycle
            ↓
必要时保留 SMMU Stage 2
            ↓
RME 系统中再执行 GPC/GPT 的 PAS 检查
            ↓
Physical Memory
```

## 8. 与当前项目的边界

当前知识库中已记录“项目没有使用 RME，但原因尚待确认”。因此本文的 GPC/GPT 部分是 RME 架构下的安全模型，**不能直接表述为当前项目已经具备的防护**。当前项目对 Full ATS 的真实限制应从 Root Complex、SMMU Stream 配置、Stage 2 使用方式和 Device 信任模型中查证。

## 面试版总结

> **Traffic tagged as Translated 不是不可伪造的授权证明。Full ATS 可能让 Device 使用最终 translation result，因此对 Device 信任和 ATS 配置要求更高；Split-stage ATS 可以让 Device 缓存 Stage 1 结果，但真实 DMA 仍由 SMMU执行 Stage 2，从而把 VM/Host 隔离保留在 Hypervisor 控制下。RME 系统还能通过 GPC/GPT 比较 transaction PAS 与目标 granule PAS，阻止跨 PAS 访问，但 GPC 不能证明该 PA 是否确实由 ATS 合法返回，也不能阻止同一 PAS 内所有越权。**

## 一句话总结

> **Full ATS 解决“少翻一次地址”；Stage 2 保住“这个上下文能访问哪些页”；GPC 保证“即使拿着 PA，也不能跨 PAS 越权”。**

## 关联记录

- [PCIe ATS 与 ATC 工作流程](35_PCIe_ATS与ATC工作流程.md)
- [Configuration Table Walk 与 GPT Walk](02_Configuration_Table_Walk与GPT_Walk.md)
- [Stage 2 Translation](14_Stage2_Translation.md)
- [SMMU TrustZone 与 Security State 机制](16_SMMU_TrustZone与Security_State机制.md)
- [当前项目未使用 RME 的原因](../Questions/2026-08-30_当前项目未使用RME的原因.md)

