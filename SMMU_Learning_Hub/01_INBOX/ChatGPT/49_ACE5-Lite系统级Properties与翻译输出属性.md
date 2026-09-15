# ACE5-Lite 系统级 Properties 与翻译输出属性

记录日期：2026-09-06  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，Training slide 中 MMU-720AE ACE-Lite TBU 的 MPAM、MTE、InvalidateHint、PBHA、RME 与 MEC Properties（培训视频截图，时间点未知）  
状态：Inbox；信号位宽、合法编码、Property 配置条件以及 MMU-720AE 对属性的生成/覆盖/透传规则待结合相关架构规范与产品 TRM 校验

## 问题

如何理解 ACE-Lite TBU interface 中的 `MPAM_Support`、`MTE_Support (Basic)`、`InvalidateHint_Transaction`、`PBHA_Support`、`RME_Support` 和 `MEC_Support`？这些属性在经过 SMMU/TBU 时分别解决什么问题？

## 回答

### 核心结论

> **现代 SMMU 的 translation result 不只是一个 PA，还可能包含资源分区、内存标签、页表硬件属性、安全 PAS 和加密上下文等系统级 transaction attributes。TBU 在完成地址翻译的同时，需要按配置生成、修改或传递这些属性，使下游各功能单元能够正确处理访问。**

## 1. 总体数据路径

这组 Property 与 Atomic、Poison、Loopback 等“transaction 怎样执行或跟踪”的能力相比，更偏向描述一次访问所属的系统上下文：

```text
Device transaction
        ↓
       TBU
        ├─ IOVA/IPA → PA
        ├─ Permission/security processing
        └─ Generate/remap/preserve output attributes
                       ↓
                      TBM
                       ↓
      Coherent Interconnect / LLC / MPAM MSC /
      GPC / Memory Encryption Engine / Memory
```

可以先按问题类型分类：

| Property | 它回答的主要问题 |
| --- | --- |
| MPAM | 这笔访问属于哪个资源分区、如何计量资源使用？ |
| MTE | 数据的 memory tag 应执行传输、更新还是匹配等操作？ |
| InvalidateHint | 相关 cache line 是否已经可以回收？ |
| PBHA | 页表为该 mapping 附带了什么 SoC 自定义硬件属性？ |
| RME | 这笔访问属于 Root、Realm、Secure、Non-secure 中哪个 PAS？ |
| MEC | 外部内存访问应使用哪个 memory-encryption context？ |

## 2. MPAM Support：资源分区与监控

MPAM 是 **Memory System Resource Partitioning and Monitoring**。它不决定“地址是否允许访问”，而是治理多个 VM/workload 共享 LLC、memory bandwidth 等资源时的分配与计量。

常见的两个核心字段概念是：

```text
PARTID → Partition ID
         标识这笔 request 属于哪个资源分区

PMG    → Performance Monitor Group
         标识监控和统计时归入哪个组
```

例如：

```text
VM A / Device A → PARTID 3
VM B / Device B → PARTID 7
                      ↓
              LLC / Memory Controller
             ├─ 按 PARTID 实施资源控制
             └─ 按 PARTID/PMG 进行监控统计
```

### MPAM 与 QoS 的区别

```text
AxQOS
→ 更接近当前 transaction 的调度优先级提示

MPAM
→ 更接近 workload 的长期资源分区与使用计量
```

二者都可能影响性能，但 QoS 不等于 MPAM 的资源治理模型。

Training slide 将 `AxMPAM` 标为 `TBM only`。在该接口配置中，可以理解为 TBU 在完成 translation/context processing 后，向系统侧输出最终 MPAM 信息；其具体来源可能与 Stream、translation context 或 SMMU 的 MPAM 配置有关，不能简单假定它来自 TBS 原样透传。

> **MPAM 不管“能不能访问”，而管“访问时可以占多少资源，以及资源使用算到谁名下”。**

## 3. MTE Support (Basic)：内存标签操作

MTE（Memory Tagging Extension）在页权限之外增加 tag 检查，用于发现地址仍位于合法 page、但 pointer 已越界或指向错误对象的情况。

```text
Pointer Logical Tag
          ↓ compare
Memory Allocation Tag
          ↓
match → 继续访问
mismatch → Tag fault
```

接口层需要携带与 tag 操作相关的信息：

| 信号 | 直观作用 |
| --- | --- |
| `AxTAGOP` | 指定当前 transaction 对 tag 执行何种操作 |
| `xTAG` | 承载 tag metadata/value |
| `WTAGUPDATE` | 指定一次写入中哪些 tag granule 需要更新 |

`AxTAGOP` 可表达 Transfer、Update、Match 等 tag operation。`WTAGUPDATE` 可以类比数据写通道的 `WSTRB`：

```text
WSTRB      → 哪些 data bytes 写入
WTAGUPDATE → 哪些 allocation tags 更新
```

Tag 是独立的 memory metadata，不要把 `xTAG` 简单理解成普通地址 bit。

从 TBU 角度看，关键是 translation 不能丢失或破坏 tag semantics；具体 Tag Check 在哪个组件执行、Basic 支持覆盖哪些操作，以及 fault 如何报告，需要结合系统 MTE 集成与 TRM 核实。

> **MTE 给数据旁边增加安全标签；`xTAG` 是标签，`AxTAGOP` 说明怎样处理，`WTAGUPDATE` 指明哪些标签需要更新。**

## 4. InvalidateHint Transaction：可回收 Cache Line 的提示

InvalidateHint 表示 requester 已不再需要相关 cache line，允许 downstream cache 回收相应容量。它通常不携带 data payload，重点在 **Hint**：

```text
Accelerator 完成 buffer 处理
              ↓
        InvalidateHint
              ↓
Downstream Cache 可以回收相关 line
```

它与 CMO 和 Deallocation Read 的边界是：

| 机制 | 核心语义 |
| --- | --- |
| CMO | 架构定义的 cache maintenance 操作，涉及明确的 clean/invalidate 要求 |
| Deallocation Read | 获取数据的同时表达对 cached copies 的处理意图 |
| InvalidateHint | 不再使用该 line 的性能/容量提示，不承担 correctness 保证 |

由于它只是 Hint，系统可以在符合协议的条件下不实际完成所有 cache 回收动作。它不能代替要求保留 Dirty data 或达到持久化点的 CMO。

Training slide 表明 `AWSNOOP` 需要扩展一位来编码 `InvalidateHint`；问答中给出的 opcode 示例为 `0b10010`。正式记录使用该编码前仍应对照当前 ACE 版本，避免把其他接口版本的编码直接套用。

> **InvalidateHint 是“这条 cache line 我不用了，你可以回收”，而不是“为了正确性必须执行一次完整 cache maintenance”。**

## 5. PBHA Support：页表携带的实现定义硬件属性

PBHA 是 **Page-Based Hardware Attributes**。它是一段与 page mapping 关联、由 translation table descriptor 提供并随翻译结果传向下游的硬件属性。

概念路径：

```text
IOVA/VA
   ↓ page-table walk
Translation descriptor
   ├─ Output Address = PA
   └─ PBHA value
          ↓
TBU output: PA + AxPBHA
          ↓
System component 按 SoC 定义解释
```

问答中把 PBHA 描述为 4-bit descriptor，但这 4 bit 的具体语义是 **IMPLEMENTATION DEFINED**。某个平台可把某种编码解释为 cache、latency 或其他 memory-system hint，不能把示例值当成 Arm 统一定义。

### PBHA 与 MPAM 的区别

```text
PBHA
→ 与 page mapping 关联
→ 由 translation descriptor 带出
→ bit 含义由 SoC 定义

MPAM
→ 与 workload/resource partition 关联
→ 具有 PARTID/PMG 等资源治理语义
```

> **PBHA 是软件借助页表给某个 mapping 贴上的硬件属性标签，但标签如何解释由 SoC 决定。**

## 6. RME Support：把安全属性扩展到四个 PAS

传统 TrustZone 主要区分 Secure 与 Non-secure。RME（Realm Management Extension）引入 Root、Realm，并形成四个 Physical Address Spaces：

```text
Root / Realm / Secure / Non-secure
```

接口使用 `AxNSE` 与既有 `AxPROT[1]` 组合编码：

| `AxNSE` | `AxPROT[1]` | Security attribute / PAS |
| ---: | ---: | --- |
| 0 | 0 | Secure |
| 0 | 1 | Non-secure |
| 1 | 0 | Root |
| 1 | 1 | Realm |

该属性会影响 GPC/GPT、memory decoding 和 security filtering 等下游处理。

`AxNSE + AxPROT[1]` 表达“这笔 transaction 属于哪个安全地址空间”，不是 read/write/execute permission：

```text
PAS/security attribute → 请求属于哪个安全世界
Page/STE/CD permission → 请求允许做什么
```

> **RME Support 用 `AxNSE + AxPROT[1]` 把 Secure/NS 二元安全属性扩展为 Root/Realm/Secure/NS 四个 PAS。**

## 7. MEC Support：内存加密上下文

MEC 是 **Memory Encryption Context**。`AxMECID` 携带 MECID，使下游 Memory Encryption Engine 能为不同安全主体选择不同的加密上下文：

```text
Realm A → MECID 5  ─┐
                    ├→ Memory Encryption Engine → External Memory
Realm B → MECID 12 ─┘
```

MECID 可以索引与 key、tweak 或其他加密状态相关的 context。确切的 context 内容、MECID 分配策略和 Point of Encryption 由系统架构及 Realm 管理软件控制。

### MECID 不等于 VMID

```text
VMID  → Stage 2 translation context / VM identity
MECID → Memory encryption context identity
```

软件可能根据 VM/Realm 关系分配 MECID，但二者协议语义不同，不能因为数值可能关联就混为一谈。

### MEC 与 GPC 的区别

```text
GPC/GPT
→ 该 requester 是否能够访问这个 PAS 中的 physical granule？

MEC
→ 该 memory transaction 在外部存储路径上使用哪个加密上下文？
```

一次 Realm 访问可以概括为：

```text
Realm transaction
       ↓
Translation → PA + Realm PAS
       ↓
GPC/GPT check
       ↓
PA + MECID
       ↓
Point of Encryption / Memory Encryption Engine
       ↓
External Memory
```

在到达 Point of Encryption 之前，片上组件之间仍可能处理 plaintext；MECID 的职责是选择外部内存保护使用的加密上下文，而不是让整个 SoC 数据路径天然变成密文。

> **GPC 管“能不能碰这块 physical granule”，MEC 管“外部内存中的数据用哪套加密上下文保护”。**

## 8. 六类属性的横向对比

| 属性 | 作用对象 | 属于哪类控制 | 典型下游消费者 |
| --- | --- | --- | --- |
| MPAM | Workload/transaction | 资源分区与监控 | MPAM MSC、LLC、Memory Controller |
| MTE | Memory granule/tag | 内存安全与错误检测 | Tag-aware memory system |
| InvalidateHint | Cache line | 性能与容量提示 | Coherent cache/interconnect |
| PBHA | Page mapping | SoC 自定义硬件属性 | Implementation-defined component |
| RME | Transaction/PAS | 安全域身份 | GPC、security filter、memory system |
| MEC | External-memory access | 加密上下文选择 | Memory Encryption Engine |

## 9. 从 SMMU/TBU 角度统一理解

```text
Translation output

PA
├─ Resource partition/monitoring  ← MPAM
├─ Tag operation/metadata         ← MTE
├─ Cache lifecycle hint           ← InvalidateHint
├─ Page-derived HW attributes     ← PBHA
├─ Physical Address Space         ← RME
└─ Encryption context             ← MEC
```

这里要区分三种 TBU 行为：

1. **Derived/generated**：某些输出属性由 STE/CD/page-table descriptor 或 SMMU 配置产生。
2. **Remapped/overridden**：某些输入属性可能按 translation context 或产品规则被转换。
3. **Preserved/forwarded**：某些 transaction 语义需要在翻译后保持并传递。

不能在没有 TRM 依据时把所有属性统称为“原样透传”，也不能说所有属性都由 TBU 自己创造。

## 10. 证据边界

- **Training slide 明确内容**：六个 Property 名称、相关信号、`TBM only` 等接口标注。
- **Arm 架构/协议背景**：MPAM、MTE、PBHA、RME PAS、MECID、InvalidateHint 的基本语义。
- **待实现核实**：当前 MMU-720AE 配置是否启用这些能力，具体属性来自哪里、如何合成/覆盖、精确信号编码、下游是否真正消费，以及当前项目未使用 RME 时相关端口如何配置。

## 一句话总结

> **现代 SMMU/TBU 输出的是“PA + 系统语义”：MPAM 管资源，MTE 管内存标签，InvalidateHint 管 cache 回收提示，PBHA 携带页表定义的实现属性，RME 标识四种 PAS，MEC 选择内存加密上下文；每项能力是否存在及属性如何生成，必须由接口 Property、translation context 和具体 SoC 集成共同确定。**

## 相关记录

- [ACE5-Lite 接口 Properties：能力声明与事务语义](48_ACE5-Lite接口Properties与事务语义.md)
- [ACE-Lite Cache Stash 与目标 Node ID / LP ID](47_ACE-Lite_Cache_Stash与目标ID.md)
- [ARM RME 扩展](10_ARM_RME扩展概述.md)
- [MMU S3 的系统定位与能力分层](43_MMU_S3系统定位与能力分层.md)
- [DPT：为 Full ATS 补充 Device/VM 级物理权限检查](41_DPT为Full_ATS补充设备权限检查.md)
