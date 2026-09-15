# SMMU 对 TrustZone 的支持

> 记录日期：2026-08-30  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 来源材料：两张培训截图，时间点未知  
> 状态：Inbox 问答整理，待结合 SMMUv3 与 MMU-720AE 规范蒸馏

## 凝练问题

**SMMU 如何将 TrustZone 的 Secure/Non-secure 状态纳入 StreamID 和地址转译体系？为什么要设置两套 StreamID namespace/Stream Table，Secure 与 Non-secure Stream 又分别能产生什么安全状态的下游 transaction？**

## 核心结论

> **在支持 TrustZone 的 SMMU 中，Secure 和 Non-secure transaction 使用独立的 StreamID namespace 和 Stream Table。SMMU 先通过 SEC_SID 确定 Stream 的安全状态，再在对应 Stream Table 中使用 StreamID 查找 Translation Context。Secure Stream 可以产生 Secure 或 Non-secure downstream transaction，而 Non-secure Stream 不能通过 translation 被“升级”为 Secure。**

## 1. 为什么需要区分 Secure 与 Non-secure Stream

CPU 有 Secure world 和 Non-secure world，设备 DMA 同样可能来自 Secure Device 或普通 Non-secure Device。

SMMU 必须先确定：

> **这笔 DMA 应使用 Secure 世界的 translation configuration，还是 Non-secure 世界的 configuration？**

因此 transaction 除了 StreamID，还具有安全状态信息。培训材料用 `SEC_SID` 表示：

```text
SEC_SID = 0 → Non-secure Stream
SEC_SID = 1 → Secure Stream
```

检查并确定该状态的过程称为：

> **SSD — Secure State Determination**

所以 SMMU translation 的入口可以理解为：

```text
Transaction
    ├─ SEC_SID
    └─ StreamID
         ↓
        SMMU
```

先确定 security state，再使用 StreamID 查找配置。

## 2. 为什么要有两套 Stream Table

支持 TrustZone 时，概念结构是：

```text
                  SMMU
                    │
             SEC_SID / SSD
              ┌─────┴─────┐
              │           │
          Secure       Non-secure
              │           │
      Secure Stream   NS Stream
          Table          Table
              │           │
             STE         STE
              │           │
      Translation    Translation
         Context        Context
```

也就是：

```text
SEC_SID=1 → 在 Secure Stream Table 中查找 SID
SEC_SID=0 → 在 Non-secure Stream Table 中查找 SID
```

因此 StreamID 数值本身不足以唯一标识一个 Stream：

```text
Stream Identity ≈ Security State + StreamID
```

## 3. 两个独立的 StreamID Namespace

`Secure SID=0` 和 `Non-secure SID=0` 可以同时存在，并表示两个完全不同的 Stream：

```text
Secure Namespace:
SID 0 → Secure GPU Context
SID 1 → Secure DMA Context

Non-secure Namespace:
SID 0 → Linux GPU Context
SID 1 → PCIe DMA Context
```

所以：

```text
SEC_SID=1, SID=0
```

与：

```text
SEC_SID=0, SID=0
```

会查找不同的 STE 和 Translation Context。

可以记成：

> **SEC_SID 先选择 StreamID 所属的安全 namespace，SID 再在该 namespace 中选择 Stream。**

## 4. Stream 的安全状态不等于输出必须保持同一状态

最重要的规则是：

```text
Secure Stream
→ 可以产生 Secure downstream transaction
→ 也可以产生 Non-secure downstream transaction

Non-secure Stream
→ 只能产生 Non-secure downstream transaction
```

原因与 TrustZone 的基本访问方向一致：

```text
Secure → 可以访问 Secure 资源
Secure → 也可以访问 Non-secure 资源
Non-secure → 只能访问 Non-secure 资源
Non-secure → 不能访问 Secure 资源
```

## 5. Secure Stream 为什么可以输出 Non-secure Transaction

假设一个 Secure DMA Engine 发出：

```text
SEC_SID = 1
SID     = 3
```

SMMU 会使用 Secure Stream Table 中对应的 STE/Context，但页表映射可以描述一个 Non-secure 输出地址：

```text
Secure DMA
   ↓
Secure Stream
   ↓
SMMU Translation
   ↓
Non-secure PA
   ↓
Non-secure Downstream Transaction
```

这是合法的，因为 Secure world 可以访问 Non-secure memory。

因此：

> **Secure Stream 表示这套 translation configuration 所属的安全控制域，并不要求所有输出 transaction 都必须是 Secure。**

## 6. Non-secure Stream 为什么不能输出 Secure Transaction

如果 Non-secure Stream 能通过修改请求属性或页表配置获得 Secure 输出，普通 OS 或不可信 PCIe Device 就可能 DMA Secure RAM，TrustZone 隔离会失效。

所以：

```text
Non-secure DMA
      ↓
     SMMU
      ↓
Secure PA / Secure Transaction   ×
```

架构必须保证 Non-secure Stream 只能产生 Non-secure downstream transaction。

培训材料还指出 `Incoming NS attribute ignored`。可以理解为：

> **已经通过 SSD 判定为 Non-secure 的 Stream，不能依赖 requester 自己提供的某个属性要求 SMMU 将其提升为 Secure。安全状态不能由不可信 requester 随意升级。**

## 7. 将 SEC_SID、SID、SSID 和 Context 串起来

```text
Transaction
    │
    ├─ SEC_SID
    │    ↓
    │  Secure / Non-secure Namespace
    │
    ├─ StreamID
    │    ↓
    │   STE
    │
    └─ SSID（如果存在）
         ↓
         CD
         ↓
Translation Context
         ↓
Stage 1 / Stage 2
         ↓
Output Security State + PA + Attributes
```

每一级分别回答：

```text
SEC_SID → 属于哪个安全域？
StreamID → 是哪个设备或 Stream？
SSID → 是该设备中的哪个进程/地址空间？
STE/CD → 使用哪套 Translation Context？
Page Table → 翻译到哪里，具有什么权限和属性？
```

## 8. 示例：相同 SID，不同安全 Stream

假设系统中有：

```text
Secure Camera DMA
SEC_SID = 1
SID     = 5

Linux GPU
SEC_SID = 0
SID     = 5
```

虽然两者 SID 都是 5，但不会冲突：

```text
Secure Camera:
SEC_SID=1
    ↓
Secure Stream Table
    ↓
STE[5]
    ↓
Secure Camera Context

Linux GPU:
SEC_SID=0
    ↓
Non-secure Stream Table
    ↓
STE[5]
    ↓
Linux GPU Context
```

## TrustZone 与 RME 的边界

这组培训内容描述的是传统 TrustZone 的 Secure/Non-secure 两安全状态模型。MMU-720AE 还可以处在支持 RME 的系统中，此时物理地址空间归属会扩展到 Root、Realm、Secure、Non-secure PAS，并涉及 GPT/GPC 等额外检查。

因此：

> **TrustZone 两世界模型是理解入口，但不能直接代表启用 RME 后的完整 PAS 模型。**

## 面试式回答

> **支持 TrustZone 时，SMMU 为 Secure 和 Non-secure traffic 维护独立的 StreamID namespace 和 Stream Table。每笔 transaction 先通过 SEC_SID 进行 Secure State Determination，选择 Secure 或 Non-secure Stream Table，再由 StreamID 选择对应的 STE 和 Translation Context。因此相同 StreamID 数值可以在两个 namespace 中表示不同 Stream。安全权限方面，Secure Stream 可以通过 translation 产生 Secure或 Non-secure downstream transaction，而 Non-secure Stream 只能产生 Non-secure transaction，防止 Non-secure requester 通过 SMMU 获得对 Secure memory 的访问权限。**

## 一句话总结

> **SEC_SID 决定“在哪个安全 namespace 查 SID”，SID 决定“使用哪个 Context”；Secure 可以访问 Non-secure，但 Non-secure 不能升级为 Secure。**

## 相关记录与资料

- [StreamID 与 Translation Context 映射](18_StreamID与Translation_Context映射.md)
- [Substream、SSID 与 PCIe PASID](19_Substream_SSID与PASID.md)
- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [ARM RME 扩展概述](10_ARM_RME扩展概述.md)
- [当前项目未使用 RME 的原因](../Questions/2026-08-30_当前项目未使用RME的原因.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

