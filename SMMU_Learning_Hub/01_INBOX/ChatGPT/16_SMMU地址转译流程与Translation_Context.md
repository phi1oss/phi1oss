# SMMU 地址转译流程与 Translation Context

> 记录日期：2026-08-30  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 状态：Inbox 问答整理，尚未独立蒸馏和验证

## 凝练问题

1. **SMMU 收到设备 transaction 后，如何完成地址转译？每一步解决什么问题？**
2. **Translation Context 是什么、包含哪些配置、如何选择和维护？它与 CPU Translation Regime 使用的寄存器有什么联系和区别？**

## 核心关系

> **SMMU translation 的主线是：先确认请求属于谁，再找到这笔请求绑定的 Translation Context，由 Context 决定是否执行 Stage 1、Stage 2、使用哪套页表和属性，最后完成权限检查并输出 PA；失败时阻止访问并报告 fault/event。**

其中：

```text
StreamID / SubstreamID
          ↓
确定 Translation Context
          ↓
决定 translation 怎么执行
          ↓
Stage 1 / Stage 2 / Bypass
```

因此，**Translation Context 是地址转译流程的“配置中枢”**。

## 一、SMMU 地址转译流程

```text
Device Transaction
  Address + StreamID + optional SubstreamID
          ↓
1. Determine Security State
          ↓
2. Translation Enabled?
   ├─ No  → Bypass
   └─ Yes
          ↓
3. Identify Translation Context
   StreamID → STE
   SubstreamID/SSID → CD（启用 Stage 1 时）
          ↓
4. Stage 1 Translation（如果启用）
   IOVA → IPA
          ↓
5. Stage 2 Translation（如果启用）
   IPA → PA
          ↓
6. Permission / Attribute / Security Check
          ↓
7. Output PA + Attributes
          ↓
   Interconnect / Memory

任一步失败
   → Terminate 或 Stall
   → 生成 Fault/Event
   → 软件处理
```

### 1. 确定 transaction 的 Security State

SMMU 首先确定这笔访问属于哪个安全上下文，具体可用状态取决于系统架构和实现。

意义是：

> **先回答“这笔访问属于哪个安全域”，从而选择适用的 SMMU 配置和保护规则。**

### 2. 判断是否启用 Translation

如果对应 Stream 配置为 bypass，SMMU 不执行正常地址转译；如果启用 translation，才继续查找 Context 并执行后续 Stage。

意义是：

> **决定 SMMU 是透传地址，还是作为地址翻译和 DMA 保护单元介入。**

### 3. 根据 StreamID/SSID 识别 Translation Context

```text
StreamID
   ↓
Stream Table
   ↓
STE
   ├─ 决定 S1/S2/Bypass 模式
   ├─ 保存 Stage 2 配置
   └─ 指向 Stage 1 CD/CD Table

SubstreamID / SSID
   ↓
选择具体 CD
   ↓
得到 Stage 1 配置
```

意义是：

> **StreamID 回答“是谁的请求”，Translation Context 回答“这笔请求应该怎么翻译”。SSID 还能区分同一设备中的不同进程或地址空间。**

### 4. 执行 Stage 1 Translation

启用 Stage 1 时：

```text
IOVA
  ↓ Stage 1
IPA
```

Stage 1 主要建立设备或 Guest 可见的虚拟地址空间。

### 5. 执行 Stage 2 Translation

启用 Stage 2 时：

```text
IPA
 ↓ Stage 2
PA
```

Stage 2 通常由 Hypervisor 控制，用来限制 Guest 或设备最终能够访问的系统物理内存。

两个 Stage 都启用时：

```text
Device IOVA
     ↓ Stage 1
Guest IPA
     ↓ Stage 2
System PA
```

可以记成：

> **Stage 1 管“设备看到什么地址空间”，Stage 2 管“该地址空间最终能访问哪些真实内存”。**

### 6. TLB Lookup 与 Table Walk

真正执行每一级 translation 时，通常先查询缓存的 translation result：

```text
Input Address
     ↓
TLB Lookup
 ┌───┴───┐
Hit     Miss
 │        ↓
 │    Table Walk
 │        ↓
 │    Fill TLB
 └───┬────┘
     ↓
Translation Result
```

意义是：

> **TLB 避免每笔 DMA 都重新读取多级页表；miss 时才需要 table walk。**

在 MMU-720AE 中，可以进一步理解为 TBU 先查询本地 translation cache，miss 后通过 DTI/LTI 路径向 TCU 请求更完整的查找或 walk。

### 7. 权限、属性和保护检查

Translation result 不只是物理地址，还包括访问权限、memory type、shareability、安全/PAS 等属性。

```text
DMA Write
   ↓
Mapping 存在
但只允许 Read
   ↓
Permission Fault
```

意义是：

> **SMMU 不仅判断“地址在哪里”，还判断“这个设备能不能以这种方式访问”。**

### 8. 输出结果或报告 Fault

成功时，SMMU 向互连输出 PA 和相关属性。失败时，SMMU 阻止错误访问，并按照配置终止或 stall transaction，同时生成 fault/event，由软件通过 Event Queue 等机制处理。

## 二、SMMU 支持的四种基本路径

```text
1. Bypass
Input → Output

2. Stage 1 only
IOVA → Stage 1 → PA

3. Stage 2 only
IPA/Input Address → Stage 2 → PA

4. Stage 1 + Stage 2
IOVA → Stage 1 → IPA → Stage 2 → PA
```

## 三、Translation Context 的概念

> **Translation Context 是 SMMU 完成某个 Stream 或 Substream 地址转译所需要的一整套配置和状态。**

它通常包括：

```text
Translation Context
├─ 是否启用 Stage 1 / Stage 2
├─ Stage 1 / Stage 2 页表基地址
├─ Translation granule
├─ Input / Output address size
├─ ASID / VMID
├─ Table-walk cacheability/shareability
├─ Translation format
├─ Permission 与属性控制
└─ 其他 translation control information
```

所以：

> **Translation Context 不等于 Page Table。Page Table 只是 Context 指向并使用的数据结构之一。**

同一个数值地址，例如 `IOVA=0x4000`，在不同 Stream、SSID、ASID 或 VMID 下可以得到不同结果；仅有地址本身不足以完成翻译。

## 四、STE 与 CD 如何组成 Context

### STE：定义 Stream 的整体翻译方向

STE 通常负责：

```text
STE
├─ Config：Bypass / S1 / S2 / S1+S2
├─ S1ContextPtr：Stage 1 CD/CD Table 的位置
├─ S2TTB：Stage 2 页表基地址
├─ S2TG / S2PS：Stage 2 granule 与地址范围
├─ S2VMID
└─ 其他 Stream 级控制
```

### CD：定义 Stage 1 的具体地址空间

CD 通常负责：

```text
CD
├─ TTB0：Stage 1 页表基地址
├─ TG0：Stage 1 translation granule
├─ IPS：Stage 1 输出地址范围
├─ ASID
├─ IR0 / OR0：table-walk cacheability
├─ SH0：table-walk shareability
└─ Valid 与其他 Stage 1 控制
```

最实用的记忆是：

> **STE 先定义这个 Stream 的“大方向”，CD 再定义 Stage 1 的具体地址空间。**

## 五、Translation Context 如何维护

SMMUv3 为了支持大量设备和进程，不为每个 Context 配置一整组固定寄存器，而是把主要配置保存在系统内存中的 STE、CD 等结构里。

```text
Software
   ↓
分配 Stream Table
   ↓
建立 STE
   ↓
分配并建立 CD / CD Table
   ↓
建立 Translation Tables
   ↓
SMMU 按 StreamID/SSID 查找并使用
```

运行时不会每次都从内存重新读取配置：

```text
StreamID / SSID
      ↓
Configuration Cache
 ┌────┴────┐
Hit      Miss
 │          ↓
 │      读取 STE/CD
 └────┬─────┘
      ↓
Translation Context
```

因此软件修改 STE/CD 后，不能只更新内存，还必须按照 SMMUv3 规定执行适用的 configuration-cache/TLB invalidation 和同步，避免 SMMU 继续使用旧 Context 或旧 translation。

## 六、与 CPU Translation Regime 的联系和区别

两者都描述一套完整的 translation environment，但服务对象、选择方式和保存位置不同。

| 对比维度 | CPU Translation Regime | SMMU Translation Context |
|---|---|---|
| 服务对象 | PE 当前执行上下文 | Device Stream/Substream transaction |
| 选择依据 | EL、Security state、当前执行状态 | StreamID、SSID/SubstreamID |
| 主要配置载体 | TTBR、TCR、MAIR、SCTLR、VTTBR、VTCR 等系统寄存器 | 系统内存中的 STE、CD 等结构 |
| Stage 1 页表基址 | TTBRn_ELx | CD.TTB0 等 |
| Stage 2 页表基址 | VTTBR_EL2 | STE.S2TTB |
| 地址空间标识 | ASID、VMID | CD 中的 ASID、STE 中的 S2VMID |
| 缓存 | TLB 和实现定义的 walk/configuration cache | TLB、Configuration Cache 等 |
| 设计倾向 | 架构定义的一类翻译环境 | 某个 Stream/进程使用的具体配置实例 |

这里是功能类比，不代表字段逐 bit 对应。

可以这样理解：

```text
Translation Regime
→ 架构层面规定一类翻译环境需要什么

Translation Context
→ 为某个 Stream/Substream 提供这些配置的一套实例
```

SMMU 采用 memory-based configuration 的重要原因是扩展性：一个 SMMU 可能同时服务大量 PCIe Function、GPU 进程、DMA engine 和虚拟机，如果每个 Context 都占用一整组固定寄存器，硬件资源难以扩展。

## 面试版总结

> **SMMU 收到设备 transaction 后，先确定安全状态并判断是否需要 translation；需要翻译时，根据 StreamID、必要时结合 SubstreamID，找到由 STE/CD 描述的 Translation Context。Context 决定 Bypass、Stage 1、Stage 2 或两级 translation，以及使用的页表、granule、地址宽度、ASID/VMID 和属性。每一级先查 TLB，miss 时执行 table walk；最终完成权限和属性检查，成功则输出 PA，失败则阻止访问并产生 fault/event。Translation Context 与 CPU Translation Regime 都是一套翻译环境，但 CPU 主要由 EL 和系统寄存器选择，SMMU 则由 StreamID/SSID 选择系统内存中的 STE/CD 配置。**

## 一句话记忆

> **Security state 决定“属于哪个安全域”，StreamID/SSID 决定“使用哪个 Context”，Context 决定“怎么翻”，Stage 1/2 决定“翻几次”，权限检查决定“能不能访问”，最终得到 PA。**

## 相关记录与资料

- [STE 与 CD 的关系](03_STE与CD的关系.md)
- [Command、Event 与 PRI Queue](05_Command_Event与PRI_Queue.md)
- [Configuration Table Walk 与 GPT Walk](02_Configuration_Table_Walk与GPT_Walk.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

