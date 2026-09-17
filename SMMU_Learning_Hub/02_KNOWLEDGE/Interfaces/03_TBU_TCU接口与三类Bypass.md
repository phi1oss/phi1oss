# MMU S3 Interfaces 主线

> 重建日期：2026-09-17  
> 状态：正式知识文档；依据培训材料重建，掌握程度待测试。  
> 唯一主线来源：`MMU.pdf` 的 **MMU S3 Interfaces** 模块，PDF 第 69–84 页（课件页 2–32）。

## 结论先行

这一模块不是让学习者背一张信号清单，而是建立 MMU S3 的完整接口分层：

```text
设备事务层：ACE5-Lite TBS/TBM 或 LTI
      │
翻译控制层：DTI-TBU / DTI-ATS message，经 BAS 传输
      │
TCU 内存访问层：QTW/DVM + 可选 PTW
      │
软件与通知层：APB5、wired interrupt、MSI、Event、PMCG
      │
组件生命周期层：Clock、Reset、LPI_CG、LPI_PD
```

理解任何接口时都要回答三个问题：

1. 它承载的是**真实数据事务、翻译控制消息，还是软件/系统管理信息**？
2. 谁是请求方、谁是响应方，数据或消息朝哪个方向流动？
3. 该接口是架构必需、产品可选，还是由构建参数决定？

---

## 1. MMU S3 接口全景

| 接口类别 | 典型接口 | 解决的问题 |
|---|---|---|
| 设备数据与本地翻译 | TBS、TBM、LTI | 设备事务如何进入/离开 TBU，紧耦合设备如何请求翻译 |
| 分布式翻译控制 | DTI-TBU、DTI-ATS over BAS | TBU/PCIe Root Port 如何与 TCU 交换翻译和维护消息 |
| TCU 内存系统访问 | QTW/DVM、可选 PTW | TCU 如何读取配置、页表和队列，以及发送 DVM/MSI |
| 软件配置 | APB5 Prog | 软件如何访问 TCU/TBU 寄存器和 Root Control Page |
| 异常与完成通知 | wired interrupt、MSI、Event | 硬件如何通知软件错误、队列变化和 CMD_SYNC 完成 |
| 性能观测 | PMCG、PMU snapshot | 如何统计并捕获 TBU/TCU 性能事件 |
| 生命周期管理 | clk、resetn、LPI_CG、LPI_PD | 如何安全地复位、门控时钟和上下电 |

这些路径彼此配合，但不能混称。例如，DTI/BAS 传送翻译控制消息，不传送设备 DMA payload；QTW/PTW 是 TCU 自己访问系统内存的 Master 接口，也不是设备数据通路。

---

## 2. 所有组件共有的全局接口

### 2.1 Clock 与 Reset

培训材料规定：

- 每个 MMU S3 组件有独立 `clk` 输入；
- 每个组件有独立 `resetn` 输入；
- TBU 和 TCU 内部实现 reset synchronization；
- BAS interconnect 组件的 reset 必须由外部同步；
- 跨时钟或跨电源域时，可以使用 ADB-400 作为异步桥。

因此，系统不能因为“这些组件属于同一个 SMMU”就假设它们同钟、同复位域。BAS 是可组合互连，CDC、reset release 和 power-domain crossing 都是集成者的责任。

### 2.2 Q-Channel Low Power Interface

- `LPI_CG`：控制 clock gating；
- `LPI_PD`：控制 power down/up，只用于 TBU 和 TCU；
- 除 BAS switch 外，培训列出的 MMU S3 组件具有 Q-Channel LPI；
- LPD-500 可以把一个 clock/power controller 连接到多个 MMU S3 组件。

`LPI_CG` 与 `LPI_PD` 是两个不同状态机：一个管理时钟，一个管理电源。进入 power control 期间反而需要保持时钟活动，以便协议握手推进。

---

## 3. ACE-Lite TBU 接口

ACE-Lite TBU 同时承担设备数据通路和翻译前端，因此其接口可以分成四组：

| 接口 | 方向/对象 | 职责 |
|---|---|---|
| TBS | 设备侧输入 | 接收尚未由 SMMU 翻译的 ACE5-Lite transaction |
| TBM | 系统侧输出 | 发出已经过 SMMU 处理的 ACE5-Lite transaction |
| 内部 LTI | BIU ↔ TLBU | BIU 向 TLBU 请求并立即使用翻译结果 |
| DTI over BAS | TLBU ↔ TCU | TLB miss、失效、同步等分布式翻译控制 |

此外还有 clock/reset、interrupt、tie-off、`LPI_CG` 和 `LPI_PD`。

### 3.1 TBS/TBM 不是简单的输入输出管脚

TBS 和 TBM 的事务身份不同：

- TBS 侧需要携带 SMMU 选择 Translation Context 所需的 `AxMMU*` 信息；
- TBU 根据翻译结果形成 TBM 输出地址和相应事务属性；
- 某些原始事务语义需要穿过 TBU继续保持，例如 Atomic、Poison、MPAM、PBHA、MTE、RME、MEC 等；
- 某些输出属性可能由 SMMU 翻译结果生成或转换，而不是机械透传输入值。

因此，“TBU 只改地址、其他位原样通过”不是可靠模型。

### 3.2 Property、Signal 与 transaction value

培训用多页表格列出 ACE5-Lite Properties。正确的阅读方法是：

- **Property**：接口实现支持某类协议能力；
- **Signal/encoding**：承载该能力所需信息；
- **transaction value**：某笔事务是否以及如何使用该能力。

支持某个 Property 不代表所有事务都启用它，也不代表一定新增一根信号；DeAllocation、CMO 等能力可以复用已有编码。

| 能力组 | 代表信息 | 主线意义 |
|---|---|---|
| SMMU 请求身份 | `AxMMUSECSID`、`AxMMUSID`、`AxMMUSSID/V` | 指明安全状态、StreamID 和可选 SubstreamID |
| Fault flow | `AxMMUFLOW`、扩展 response bit | 管理 Translation Fault 的事务响应 |
| Cache stash | `AWSNOOP`、目标 Node/LPID 及有效位 | 保持 stash transaction 的目标信息 |
| 原事务语义 | Atomic、Loopback、Poison、Unique ID、Read Data Chunking、CMO | 防止翻译路径破坏上游协议语义 |
| 输出资源/安全属性 | MPAM、MTE、PBHA、RME、MEC | 将翻译或上下文相关属性带到系统侧 |
| 活动唤醒 | `AWAKEUP` | TBM 有活动时驱动低功耗唤醒条件 |

精确信号编码和合法组合需要相应 AMBA Specification；培训表格用于建立“哪些信息必须跨越翻译边界”的概念地图。

### 3.3 AxMMUVALID：per-transaction physical bypass

`AxMMUVALID` 用于受信任路径逐事务说明输入地址是否需要 SMMUv3 Translation：

- `AxMMUVALID == 1`：该事务具备正常 SMMU translation 所需的 `AxMMU*` 信息，translation **可能**进行；
- `AxMMUVALID == 0`：输入必须已经是 PA，SMMUv3 Translation 和属性转换被旁路，其他 `AxMMU*` 信息无效；
- 培训称其为 **NoStreamID transaction**；
- 即使旁路 SMMUv3 Translation，培训仍要求执行 GPC。

LTI 上对应的是 `LAMMUV`。它和 Global Bypass、STE Stream Bypass 不同：后两者仍可能按照 SMMU 配置进行属性处理，而 NoStreamID 是事务入口处的 Physical Bypass。

### 3.4 AxUSER 的输入输出职责不同

- TBS `AxUSER` 可以包含 TLBLOC：`mtlbidx`、`mtlbway`、`mtlbpart`，用于直接索引或分区相关信息；
- TBM `AxUSER` 可以包含 Outer Cacheable hint；
- 两侧都可以有用户自定义位；
- 信号宽度取决于 TBU 构建参数。

这说明 TBS 与 TBM 的 `AxUSER` 不是同一段完全透明的 sideband。标准定义区域和 user-defined 区域由配置共同决定。

---

## 4. LTI TBU 接口

LTI TBU 面向 PCIe Root Port 或其他 LTI I/O 组件，只提供翻译前端，不承担 ACE-Lite 数据 payload：

```text
LTI device
  → 一个或多个 LTI channel
  → LTI TBU / TLBU
      ├─ TLB hit：LTI response 立即供当前事务使用
      └─ miss：DTI-TBU over BAS → TCU
```

培训给出的产品特征包括：

- 最多支持 8 个 LTI interface；
- 两个 Virtual Channel：non-posted/read 与 posted/write；
- 不实现 LTI user signals；
- ID、SID、ordering group、TLBLOC、loopback、MECID 等宽度由 `TBUCFG_*` 构建参数决定；
- 同样具有 DTI、interrupt、clock/reset、tie-off、`LPI_CG` 和 `LPI_PD`。

LTI response 不能由请求方缓存，并不否认 LTI TBU 内部的 TLBU 可以缓存翻译。前者是接口使用规则，后者是 TBU 的本地实现能力。

---

## 5. TCU 的接口地图

TCU 既是 DTI Manager，也是配置/页表访问者和软件控制中心。

### 5.1 QTW/DVM：默认的综合内存系统接口

培训列出的 QTW/DVM 流量包括：

- Configuration Table Walk；
- Translation/Page Table Walk；
- HTTU；
- VMS fetch；
- GPT walk；
- DPT walk；
- Secure/Non-secure/Realm Command Queue read；
- Secure/Non-secure/Realm Event Queue write；
- Non-secure/Realm PRI Queue write；
- DVM TLB invalidation message；
- MSI。

之所以称为 QTW/DVM，是因为它不只“读页表”：它同时承担配置表、队列、保护表、DVM 和通知等多种 TCU 内存系统流量。

### 5.2 可选 PTW：为高 PTW 带宽分流

可选 ACE5-Lite PTW interface 用于 Translation Table Walk 和 HTTU，面向 PTW 带宽非常高的系统。根据本模块课件，是否存在专用 PTW 决定这些流量是否从综合 QTW/DVM 路径分离。

培训页还把 DPT walk 列在 QTW/DVM 清单中，而 DPT 又是 MMU S3 r1 能力。具体产品是否实现 DPT、DPT 流量最终走哪个端口，必须以相应 revision 的 TRM 为准，不能只用该概览页推断。

### 5.3 其他 TCU 接口

| 接口 | 职责 |
|---|---|
| DTI over BAS | 与 TBU 和 PCIe Root Port 通信 |
| Dedicated MSI over BAS | 发送 TCU 生成的 AXI5-Stream MSI |
| APB5 Prog | 软件配置 TCU/TBU registers，并访问 Root Control Page |
| LPI_CG / LPI_PD | Clock gating 和 power down/up |
| Interrupt/Event/PMU snapshot | 错误、完成和性能观测 |

APB5 是软件寄存器入口，不用于读取 STE/CD/PTE 内容；这些内存结构仍由 TCU 通过相应内存系统接口访问。

### 5.4 TCU interface Properties

QTW/DVM 与 PTW 都可能支持 Atomic、Poison、Unique ID、Wakeup、MPAM、PBHA、RME 和 MEC 等 Properties。QTW/DVM 还额外承担 DVM v8.4/v9.2、coherency connection 和 AC-channel wakeup。

这意味着 TCU 自身的 page/configuration walk 也必须带有正确的系统属性；不能把这些 Property 误认为只属于被翻译的设备 transaction。

---

## 6. Power、Clock 与 DTI Connection 的依赖关系

### 6.1 Power down

- 每个 TBU 断电前，必须先与 TCU disconnect；
- TCU 断电前，所有 TBU 和 PCIe Root Port 都必须 disconnect；
- 相应 Q-Channel 必须进入 `Q_STOPPED`，之后才能 gate clock 或移除 power；
- power control 进行期间必须保持 clock active，`qactive_cg` 被拉高并拒绝 LPI_CG clock-gating request。

### 6.2 Power up

- TCU 必须先上电；
- 然后 TBU/PCIe Root Port 才能上电并建立 connection；
- TBU 和 TCU 的状态在断电后不保留；
- 每次重新上电后都需要重新编程 SMMU registers。

因此，低功耗不是单独的 Q-Channel 问题。安全顺序要同时满足 DTI connection lifecycle、Q-Channel 状态和 register restore：

```text
停止新工作 → 排空/断开 DTI → Q_STOPPED → 断电
TCU 上电并恢复配置 → requester 上电 → 建立 DTI 连接 → 恢复工作
```

---

## 7. Interrupt 与 Event 必须分开

### 7.1 TBU Interrupt

TBU 可以提供：

- RAS FHI/ERI/CRI 的 edge-triggered 或 level-sensitive wired interrupt；
- PMU counter overflow；
- 无法连接 TCU（connection denial 或 token 不足）产生的 `crit_err`。

TBU 自身不能生成 MSI；ACE-Lite TBU 可以把上游 ACE-Lite MSI 从 TBS 传到 TBM。LTI device 应具有自己的 MSI interface，直接连接 GIC ITS。

### 7.2 TCU Interrupt

TCU 的通知对象更多，包括：

- Event Queue 从空变为非空；
- CMD_SYNC completion；
- Global Error；
- PRI Queue；
- PMU overflow；
- GPF、GPT configuration fault；
- RAS FHI/ERI/CRI。

这些中断始终可以采用 edge-triggered 或 level-sensitive wired signal。Queue/Global Error 还可以使用 MSI；MSI 可以从 dedicated BAS MSI interface 发出，也可以通过 QTW/DVM 发 ACE5-Lite MSI，具体取决于 GIC/ITS 集成。

### 7.3 CMD_SYNC：Interrupt completion 与 Event completion

软件等待 CMD_SYNC 时有两种唤醒机制：

| 机制 | 硬件路径 | CPU 行为 |
|---|---|---|
| Interrupt | wired interrupt 或 MSI → GIC | CPU 唤醒并进入 exception handler |
| Event | TCU Event interface → Event network | CPU 从 WFE 唤醒并继续执行，不切换到 interrupt context |

Event interface 使用 `eventoreq/eventoack` 四阶段握手，`eventoreq` 保持为高直到收到 acknowledgement。具体使用哪种 completion signal 由 `CMD_SYNC.CS` 选择。

所以“event”不是 EVTQ entry 的同义词，也不是“没有中断线的中断”。它是一条用于 WFE 唤醒的独立硬件通知路径。

---

## 8. Performance Monitoring 与 PMU Snapshot

MMU S3 实现 SMMUv3 Performance Monitor Extension：

- 每个 TBU 和 TCU 有独立 PMCG；
- 可以计数架构和微架构事件；
- counter 可以按 StreamID filter；
- 实时值由 `SMMU_PMCG_EVCNTRn` 读取；
- snapshot value 保存在 `SMMU_PMCG_SVRn` shadow register。

### 8.1 Snapshot 的两种触发方式

- 硬件：`pmusnapshot_req/ack` 四阶段异步握手，通常由 CTI 等 debug infrastructure 控制；
- 软件：通过 PMCG capture control 触发。

培训第 80 页把软件触发字段写成 `SMMU_PMCG_CFGR.CAPTURE`；当前资料库的架构/TRM校正表明能力查询与触发寄存器需要区分，正式编程应查对应产品规范。详细说明见 [PMCG 与 PMU Snapshot](../Debug/03_PMCG与PMU_Snapshot.md)。

Snapshot 的价值是得到一个 PMCG 内多个 counter 的一致观察点；它不自动证明多个不同 clock domain 的 PMCG 在同一个物理周期采样。

---

## 9. DTI：分布式组件之间的消息协议

DTI 是 message-based protocol，BAS 是 Arm SMMU 中承载 DTI message 的物理协议。

### 9.1 两个 sub-protocol

| Sub-protocol | 通信双方 | MMU S3 r0 支持范围 |
|---|---|---|
| DTI-TBU | TBU ↔ TCU | 内部使用 DTI-TBUv3；TCU 不支持 v1/v2 TBU |
| DTI-ATS | PCIe Root Port ↔ TCU | 外部支持 v1、v2、v3 ATS Manager |

DTI message 按功能可以分为 connection/disconnection、translation request/response、invalidation/synchronization、page request 和 register access。

版本兼容必须按 sub-protocol 判断，不能只说“支持 DTI v3”就推断所有 v1/v2 endpoint 都不兼容。

### 9.2 Token flow control

Token 用于限制在途工作量：

- Translation token 限制 outstanding translation request；
- Invalidation token 限制 outstanding invalidation；
- request 消耗 token；
- response/acknowledgement 归还 token；
- Translation token 由 TCU 分配；
- Invalidation token 由 TBU 或 PCIe Root Port 授予；
- disconnect 时归还相应 token 资源。

TBU 的 `max_tok_trans` 连接参数不能请求超过 TCU translation slot 的数量。Token 是流控容量，不是请求身份、排序号或完成语义。

---

## 10. BAS：可组合的 DTI 物理互连

MMU S3 不固定内部 BAS 拓扑，各组件由集成者分别实例化并连接。

| 组件 | 作用 |
|---|---|
| BAS sizer | 上/下调数据宽度，以性能换取布线数量 |
| BAS switch | 把多个 DTI subordinate（TBU/PCIe Root Port）汇聚到一个 DTI manager（TCU） |
| BAS register slice | 切断长时序路径 |
| ADB-400 | 跨 clock/power domain 的异步桥 |

培训给出的接口宽度是：downstream 固定 160 bit；upstream 根据 MECID width 为 160 或 192 bit。该数值属于本模块描述的 MMU S3 配置，不应推广为所有 DTI/BAS 实现的固定宽度。

### 10.1 BAS switch 为什么要重映射 TID/TDEST

每个 subordinate interface 都拥有自己的局部 ID namespace。进入共享 Manager 侧后必须避免不同端口出现相同 TID：

- Downstream：switch 把各 subordinate 的局部 TID 扩展为 Manager 侧全局唯一 TID；
- Upstream：根据 Manager 侧 TDEST 找到目标 subordinate，并缩减回该端口的局部 namespace；
- `DECMIN_SIn/DECMAX_SIn` 为每个 subordinate 分配互不重叠的全局范围；
- `ID_WIDTH` 必须容纳最大的全局编码。

培训示例分配：

| Port | Local range | Global range |
|---|---:|---:|
| S0 | 0–5 | 0–5 |
| S1 | 0–3 | 6–9 |
| S2 | 0–3 | 10–13 |
| S3 | 0–7 | 14–21 |

最大编码是 21，需要表示 0–21 共 22 个值，因此示例的 `ID_WIDTH` 至少为 **5 bits**。

---

## 11. 端到端串联

以 ACE-Lite TBU 出现 TLB miss 为例：

```text
1. Device transaction 从 TBS 进入，携带 SID/SSID 等 AxMMU* 信息
2. BIU 通过内部 LTI 向 TLBU 请求翻译
3. TLBU miss，经 DTI-TBU/BAS 向 TCU 发 Translation Request
4. TCU 通过 QTW/DVM 或可选 PTW 访问配置/翻译表
5. Translation Response 经 BAS/DTI 返回 TLBU，并受 token 流控
6. BIU 形成 TBM 输出地址和适用的事务属性
7. Fault/完成按需通过 wired interrupt、MSI 或 Event 通知软件
8. PMCG 可以统计该路径上的相应事件
9. 若组件进入低功耗，必须先完成 DTI disconnect 与 Q-Channel 停止流程
```

这条流程把模块中的接口、Properties、流控、通知和电源管理放回同一条因果链。

---

## 12. 培训材料的使用边界

1. **模块边界**：本篇只使用 PDF 第 69–84 页；旧文档中的第 41、87–88 页和历史问答推导不再作为本篇主线。
2. **Property 是构建能力**：表格列出的是可支持能力，不代表当前实例全部启用，也不代表每笔事务都使用。
3. **AxMMUVALID 只适用于可信 Physical Bypass**：它不是任意设备绕过隔离的通用开关。
4. **QTW/PTW 流量划分存在版本边界**：培训概览与具体产品 revision 可能不同，尤其是未来 DPT 相关流量。
5. **AMBA interface summary 标注 r0 only**：协议 issue 和 DTI 版本不能自动推广到 r1 或其他产品。
6. **培训 PMU 软件触发字段需校正**：正式代码应依据架构/TRM核对 CAPR/CFGR，而不是照抄课件字段名。
7. **BAS 宽度和端口数是产品配置**：不能从课件示例推导当前 SoC 的实例参数。

## 面试版总结

> MMU S3 的接口可以分为五层：TBS/TBM或LTI承载设备事务与本地翻译，DTI消息通过BAS连接TBU/Root Port与TCU，QTW/DVM和可选PTW让TCU访问内存结构，APB/Interrupt/MSI/Event/PMCG承担软件控制与观测，Clock/Reset/Q-Channel负责组件生命周期。判断任何接口时，先确认它传的是数据、翻译消息还是管理信息。

## 相关专题

- [ACE5-Lite Properties 与输出属性](04_ACE5-Lite_Properties与输出属性.md)
- [SMMU 低功耗、复位与连接顺序](../Integration/01_SMMU低功耗_复位与连接顺序.md)
- [PMCG 与 PMU Snapshot](../Debug/03_PMCG与PMU_Snapshot.md)
- [DTI 协议、连接与 Token 流控](05_DTI协议_连接与Token流控.md)
- [BAS 互连与 TID/TDEST 映射](06_BAS互连与TID_TDEST映射.md)

## 来源

- [MMU.pdf：MMU S3 Interfaces](../../99_SOURCE/Training/MMU.pdf)，PDF 第 69–84 页。

