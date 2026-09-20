# MMU S3 Translation 主线

> 整理日期：2026-09-20  
> 状态：正式知识文档；依据培训材料整理，掌握程度待测试。  
> 唯一主线来源：`MMU.pdf` 的 **MMU S3 Translation** 模块，PDF 第 94–103 页（课件页 2–20）。

## 结论先行

MMU S3 的完整 Translation 不是直接拿输入地址查页表，而是一条由安全状态约束的选择与保护链：

```text
请求携带 SEC_SID、StreamID、可选 SubstreamID 和输入地址
        │
        ├─ SEC_SID：选择 Non-secure / Secure / Realm 资源 bank
        ├─ StreamID：查 STE，取得 Stream 模式和 Stage 2 配置
        ├─ SubstreamID：查 CD，取得 Stage 1 配置（若适用）
        │
        ├─ Stage 1：VA → IPA
        ├─ Stage 2：IPA → PA
        │
        └─ GPC：检查输出 PAS 是否允许访问目标 PA granule
                         ├─ Pass：形成 translation response / 继续访问
                         └─ Fault：报告 Translation Fault、GPF 或 GPT lookup error
```

因此，本模块有三条相互依赖但不能混为一谈的主线：

1. **Security**：请求使用哪套翻译资源、最多能够输出到哪些 PAS。
2. **Translation**：如何从 STE/CD 找到页表，再完成 Stage 1/Stage 2 lookup。
3. **Protection**：翻译得到 PA 后，GPT/GPI 是否允许该安全状态访问目标物理 granule。

---

## 1. Translation Security 与 Protection Security

### 1.1 RME 的四种 Security State

RME 把 TrustZone 模型扩展为：

- **Non-secure**：由 Non-secure software 控制；
- **Secure**：由 Secure software 控制；
- **Realm**：由 Realm software 控制；
- **Root**：由 Root software 控制。

但“系统存在四种 Security State”不等于 SMMU 支持四种 Translation：

| 能力 | 支持的 Security State | 原因 |
|---|---|---|
| Translation | Non-secure、Secure、Realm | I/O device 不归 Root 所有，因此没有 Root Translation |
| Granule Protection | Non-secure、Secure、Realm、Root | GPT 由 Root 控制，必须能够描述四种 PAS 的访问许可 |
| DPT | Realm Translation | 培训材料限定为 Realm 路径，且属于 MMU S3 r1 能力 |

Root 虽然不发起设备 Translation，却控制 GPT、GPC 和相关 Root register。不能把“没有 Root Translation”误解为“Root 不参与 SMMU”。

### 1.2 Translation Security 限制输出 PAS

培训材料给出的输出限制是：

| Translation Security | Non-secure PAS | Secure PAS | Realm PAS | Root PAS |
|---|:---:|:---:|:---:|:---:|
| Non-secure | ✓ | ✗ | ✗ | ✗ |
| Secure | ✓ | ✓ | ✗ | ✗ |
| Realm | ✓ | ✗ | ✓ | ✗ |

这张表表达的是 Translation 所允许形成的 PAS 范围；最终物理访问仍要接受 GPC。即使某类 Translation 可以输出到某 PAS，也不代表任意 PA granule 都允许访问。

---

## 2. Resource Banking：安全状态先选择资源域

SMMU 把以下资源按 Non-secure、Secure、Realm 分 bank：

| 资源 | Banking 关系 |
|---|---|
| Registers | `SMMU_*`、`SMMU_S_*`、`SMMU_R_*` |
| Configuration structures | 各自的 Stream Table 和 Context Descriptor Table |
| Command/Event Queue | 三个安全状态各自拥有 |
| PRI Queue | Non-secure 和 Realm 拥有；Secure 没有 |
| Interrupts | 随对应安全资源分开 |
| Performance counters | 按 Security bank 管理 |

此外，`SMMU_ROOT_*` register 专门用于 GPT 控制与失效。

Banking 的设计目的不是复制同一套数据方便访问，而是防止一个安全状态通过共享配置或队列污染另一个安全状态的翻译控制面。

---

## 3. Secure State Determination（SSD）

### 3.1 SEC_SID 决定 Translation Security

请求进入 TBU 时，用 SEC_SID 选择 Translation Security：

| 接口 | 信号 |
|---|---|
| ACE-Lite TBU 的 TBS | `AxMMUSECSID` |
| LTI TBU 的 LTI interface | `LASECSID` |

编码为：

| SEC_SID | Translation Security |
|---:|---|
| `0b00` | Non-secure |
| `0b01` | Secure |
| `0b10` | Realm |

选定 Security State 后，后续 `SMMU_(*_)STRTAB_BASE`、Stream Table、CD Table、Queue 和其他资源都来自对应 bank。

### 3.2 Translation Security 不一定等于输入事务的 Security attribute

培训强调，SEC_SID 可以与输入 transaction 的 device Security 独立。在常见连接中，它由 `AxNSE/AxPROT[1]` 或 `LANSE/LAPROT[1]` 组合生成；也可以由系统集成单独控制或固定 tie-off。

这意味着两组信息回答不同问题：

- Incoming Security attribute：该总线事务当前带有什么安全属性；
- SEC_SID：SMMU 应使用哪套 Translation Security 和 banked resource。

PCIe transaction 按培训要求只使用 Non-secure 或 Realm Translation，不使用 Secure Translation。

---

## 4. Register Ownership 与 Override

### 4.1 默认所有权

- Secure register 通常只允许 Secure 或 Root software 访问；
- Root register 通常只允许 Root software 访问；
- APB5 的 `pnse_prog` 和 `pprot_prog` 表示 programming transaction 的 Security。

### 4.2 SCR：允许 Non-secure 管理指定 Secure 功能

`TBU_SCR/TCU_SCR` 只能由 Secure 或 Root software 访问，但其 feature control 可以把特定 Secure 功能的 register ownership 下放给 Non-secure，例如：

- `SMMU_S_INIT` 的 cache/TLB invalidation；
- RAS error control/reporting；
- TBU/TCU microarchitectural register。

Reset 默认值由 `sec_override` tie-off 决定。这里的 feature 是按功能选择的权限覆盖，不应理解成一位开关永久开放全部 Secure bank。

### 4.3 RCR：允许较低状态管理指定 Root 微架构功能

`TBU_RCR/TCU_RCR` 只能由 Root software 访问，其 feature control 可以把指定 Root microarchitectural register 的访问权下放给 Secure；若对应 SCR 也继续下放，则最终可以由 Non-secure 管理。

培训用下表概括逐 feature 的 ownership override：

| SCR.feature | RCR.feature | NS register | Secure register | Realm register | Root register |
|---:|---:|---|---|---|---|
| 0 | 0 | Non-secure | Secure | Realm | Root |
| 0 | 1 | Non-secure | Secure | Realm | Secure |
| 1 | 0 | Non-secure | Non-secure | Non-secure | Root |
| 1 | 1 | Non-secure | Non-secure | Non-secure | Non-secure |

这张表描述的是 register ownership，不会改变某笔设备请求的 SEC_SID，也不会把 Non-secure Translation 变成 Realm Translation。

---

## 5. Configuration Lookup：先找翻译环境

Translation 的第一阶段不是读 PTE，而是取得 Translation Context。

### 5.1 选择安全资源 bank

```text
SEC_SID
  → Non-secure / Secure / Realm bank
  → 选择该 bank 的 STRTAB_BASE、STRTAB_BASE_CFG 和 Stream Table
```

### 5.2 SID 选择 STE

- `SMMU_(*_)STRTAB_BASE` 给出 Stream Table base address；
- `SMMU_(*_)STRTAB_BASE_CFG` 给出 Stream Table format 和 size；
- StreamID 索引 Stream Table；
- STE 给出 Stream 的 Translation mode 和 Stage 2 Translation configuration；
- STE 还提供 Stage 1 CD Table 入口（若使用 Stage 1）。

### 5.3 SSID 选择 CD

- `STE.S1ContextPtr` 给出 CD Table base；
- `STE.S1Fmt/S1CDMax` 给出 CD Table format 和 size；
- SubstreamID 索引 CD Table；
- CD 给出 Stage 1 Translation configuration 和 Translation Table base。

因此职责关系是：

```text
SEC_SID 选择安全 bank
SID     选择 Stream/STE
SSID    选择 Stage 1 Context/CD
ASID    标识所选 Stage 1 地址空间并参与相关缓存管理
```

这些标识不能互换；Configuration Lookup 也不能简化成“用 SID 直接找到 PTE”。

---

## 6. Translation Lookup：再解析地址映射

STE/CD 提供页表入口和使用规则后，SMMU 才进入 Translation Lookup：

1. 读取当前 Translation Table descriptor；
2. Table descriptor 给出下一级 Table base；
3. Block/Page descriptor 给出 Translation result 和权限/属性；
4. Stage 1 输出 IPA；
5. Stage 2 把 IPA 转换为最终 PA；
6. 使用所得 Translation information 形成 response 或继续 client transaction。

培训图展示的是完整 memory lookup 路径。实际硬件允许缓存 STE、CD、S1/S2 Translation 和 GPT entry；命中时不会重新执行图中的全部 memory read。

### 6.1 Fault 可以发生在两个 lookup 阶段

- Configuration Lookup fault：例如 Stream/CD 配置无效或找不到合法上下文；
- Translation Lookup fault：例如 descriptor invalid、权限不允许或地址超范围。

出现 Translation-related fault 后，transaction 可以：

- **Terminate**：结束并报告 fault；
- **Stall**：保存 transaction，等待软件修复映射后重试。

Stall fault model 允许页面不长期 pinned，并支持 dynamic paging。它表示硬件能暂停并重试相关 transaction，不表示 SMMU 自己创建页面或修改操作系统页表。

---

## 7. Lookup Process 图如何计算

培训第 99 页给出一个特定 worst-case：

- 2-level Stream Table：STE L1 + STE L2；
- 2-level CD Table：CD L1 + CD L2；
- 4-level Stage 1 Table；
- 4-level Stage 2 Table；
- 每次读取一个 Stage 1 table level 前，都需要把该 table address 经 Stage 2 转为 PA；
- Stage 1 得到最终 IPA 后，还需要一次 Stage 2 walk 得到输出 PA。

### 7.1 为什么会出现多次 Stage 2 Walk

Guest 管理的 Stage 1 page table 位于 IPA space。SMMU 要读取 S1 L0/L1/L2/L3 descriptor，必须先用 Stage 2 找到每一级表所在的 PA。最后，S1 产生的目标 IPA 还要再经过一次 Stage 2。

```text
Configuration lookup：2 STE + 2 CD = 4 次 descriptor read

Stage 1 walk：4 次 S1 descriptor read

Stage 2 walks：
  4 次，用于定位 S1 L0/L1/L2/L3 descriptor 所在 PA
  1 次，用于把最终 S1 output IPA 转成 PA
  共 5 个 S2 walk × 每个 4 level = 20 次 S2 descriptor read

总计：4 + 4 + 20 = 28 次 memory descriptor read
```

可以写成：

```text
总 lookup 数 = C + L1 + (L1 + 1) × L2

C  = Configuration descriptor read 数
L1 = Stage 1 level 数
L2 = Stage 2 level 数
```

本图代入 `C=4, L1=4, L2=4`，得到 28。

这个数字不是每笔访问的固定成本：Linear/2-level table 选择、起始 level、Block descriptor、Stage 配置以及各级 cache hit 都会显著减少实际 memory access。

---

## 8. GPC：Translation 之后的物理地址空间保护

Granule Protection Check（GPC）检查 translated transaction 的 Security attribute 是否允许访问目标 PA 所属 granule。

它解决的是：

```text
Translation 已经得到了 PA
        ↓
该请求的 outgoing PAS 是否允许访问这个 PA granule？
        ↓
GPT/GPI → Pass 或 Fault
```

GPC 使物理页面能够在不同 PAS 之间动态分配，而不必完全依赖固定 memory carveout 和 completer-side filter。培训明确指出 MMU S3 不支持 GPC elision，即不能因为某类路径看似可信就跳过要求的检查。

### 8.1 哪些访问需要 GPC

**SMMU-originated access：**

- CTW/PTW；
- HTTU；
- DPT walk（r1）；
- Command/Event/PRI Queue access；
- MSI write。

**Client-originated access：**

- 对最终 translated PA 执行 GPC。

这说明 GPC 不只保护设备最终数据访问，也保护 SMMU 自己读取配置、页表和队列时产生的 memory access。

---

## 9. GPT 与 GPI

### 9.1 GPT 回答的问题

Granule Protection Table（GPT）是内存中的查找表，为每个 physical granule 给出 Granule Protection Information（GPI）：

```text
输入：PA + 请求的 outgoing PAS
GPT lookup：目标 granule 的 GPI
输出：该 PAS 是否允许访问
```

GPT 最多需要两级 walk，GPT entry 可以缓存。GPT fetch 本身使用 Root PAS，并且不再接受 GPC，否则会形成“为了检查 GPT fetch 又需要读取 GPT”的递归依赖。

培训还说明，为 Translation Table read 准备的 GPT fetch 可以被 speculative execution；这只说明允许提前取保护信息，不代表最终不需要使用正确 GPI 完成检查。

### 9.2 Descriptor 与 GPI 编码

- L0 Table descriptor：指向 L1 GPT；
- L0 Block descriptor：直接携带适用于较大区域的 GPI；
- L1 Granules/Contiguous descriptor：包含多个 granule 的 GPI field；
- 每个 GPI field 为 4 bits。

培训列出的 GPI：

| GPI | 许可 |
|---:|---|
| `0b0000` | No access |
| `0b1000` | Secure PAS only |
| `0b1001` | Non-secure PAS only |
| `0b1010` | Root PAS only，仅适用于 NoStreamID transaction |
| `0b1011` | Realm PAS only |
| `0b1111` | All PAS，后续依赖 completer-side filtering |

`0b1111` 不等于“完全不保护”：它表示 GPT 不按 PAS 阻止访问，系统仍应使用 completer-side filtering 执行相应目标端保护。

---

## 10. GPT Walk 的地址切分

培训使用三个参数定义 GPT lookup：

- `t`：Protected Physical Address Size（PPS），来自 `SMMU_ROOT_GPT_BASE_CFG.PPS`；
- `s`：Level 0 GPT entry size，来自 `L0GPTSZ`；
- `p`：Physical Granule Size（PGS），来自 `PGS`。

物理地址切分为：

```text
PA[51:t]      超出 protected physical address range 的高位
PA[t-1:s]     L0 index
PA[s-1:p+4]   L1 index
PA[p+3:p]     GPI index
PA[p-1:0]     granule 内 offset
```

当 L0 entry 是 Table descriptor 时，先找到 L1 table；L1 Granules descriptor 包含 16 个 GPI field，`PA[p+3:p]` 在其中选择一个 4-bit GPI。

这里的 `p` 决定 physical granule 大小，不是 CPU Translation Table 的 TG0/TG1；GPT 的 granule protection 与 VA page translation 是两套不同的表和索引规则。

---

## 11. GPF 与 GPT Lookup Error 的区别

### 11.1 Granule Protection Fault（GPF）

GPT lookup 能够给出保护结论，但访问不被允许：

- PAS mismatch；
- Secure、Realm 或 Root access 超出 PPS。

报告路径：

- 写入 `SMMU_ROOT_GPF_FAR`；
- 触发 `gpf_far` interrupt。

### 11.2 GPT Lookup Error

硬件无法正常取得有效 GPI：

- `SMMU_ROOT_GPT_BASE_CFG` 非法；
- L0/L1 base 超出 PPS；
- GPT descriptor invalid；
- GPT fetch 遇到 External Abort/SLVERR；
- GPT fetch 遇到 RAS error。

报告路径：

- 写入 `SMMU_ROOT_GPT_CFG_FAR`；
- 触发 `gpt_cfg_far` interrupt。

诊断时应先区分：GPF 是“有效保护策略拒绝访问”，GPT lookup error 是“保护表配置或读取过程本身失败”。

---

## 12. 端到端 Translation Flow

```text
1. TBU 接收请求：SEC_SID + SID + 可选 SSID + input address
2. SEC_SID 选择 Non-secure/Secure/Realm resource bank
3. SID 查 STE，获得 Stream mode、Stage 2 配置和 CD Table 入口
4. SSID 查 CD，获得 Stage 1 配置和 S1 table base
5. 查询已缓存的 configuration/translation；未命中则执行 memory lookup
6. Stage 1：VA → IPA（若启用）
7. Stage 2：IPA → PA（若启用）
8. 根据 Translation Security 形成允许的 outgoing PAS
9. GPT lookup 得到目标 granule 的 GPI
10. GPC 比较 outgoing PAS 与 GPI
11. Pass：返回 Translation result / 发出访问
    Fault：terminate 或 stall，并走相应 fault report 路径
```

这条链说明“地址算对”并不等于“访问一定被允许”：Translation Table 决定地址和访问属性，GPT 再决定该物理 granule 对相应 PAS 是否开放。

---

## 13. 培训材料的使用边界

1. **模块边界**：本篇只使用 PDF 第 94–103 页，不混入后续 Caching/Prefetch/Hazarding 模块。
2. **RME 能力需要产品确认**：培训描述 MMU S3 系列能力，不能自动证明当前实例启用了 Realm、Root register 或 GPT。
3. **Lookup 图是 worst-case miss path**：28 次只在指定的 2-level ST/CD、4-level S1/S2 且全部需要 memory lookup 时成立。
4. **Root 无 Translation 不等于无 Root 功能**：Root 仍控制 GPT/GPC 和相关寄存器。
5. **Register override 不改变 transaction security**：SCR/RCR 改的是指定 register 的 software ownership，不改 SEC_SID。
6. **DPT 仍是 r1/Realm 边界**：本模块提到 DPT walk 和 DPT check 时，必须结合具体 MMU S3 revision 判断。
7. **GPT 与 Translation Table 分工不同**：前者管理 PA granule 的 PAS许可，后者管理 VA/IPA 到 PA 的映射与访问属性。

## 面试版总结

> MMU S3 先用 SEC_SID 选择 Non-secure、Secure 或 Realm 资源 bank，再用 SID/SSID 找到 STE/CD 和 Stage 1/2 页表；地址翻译得到 PA 后，还要通过 GPT/GPI 做 GPC，确认该安全状态允许访问目标物理 granule。Translation 解决“地址到哪里”，GPC 解决“这个 PAS 能否访问那里”。

## 相关知识

- [AArch64 多级页表与地址翻译](01_AArch64多级页表与地址翻译.md)
- [AArch64 Translation Regimes](02_AArch64_Translation_Regimes.md)
- [Translation Descriptors 与 Memory Attributes](03_Translation_Descriptors与Memory_Attributes.md)
- [Programming the SMMU](../Software/02_Programming_the_SMMU主线.md)
- [MMU S3 Introduction and Topology](../Architecture/01_MMU_S3组件与设备集成拓扑.md)
- [MMU S3 Interfaces](../Interfaces/03_TBU_TCU接口与三类Bypass.md)

## 来源

- [MMU.pdf：MMU S3 Translation](../../99_SOURCE/Training/MMU.pdf)，PDF 第 94–103 页。

