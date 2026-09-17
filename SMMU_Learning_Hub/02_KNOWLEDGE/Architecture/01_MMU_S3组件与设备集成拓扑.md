# MMU S3 Introduction and Topology 主线

> 重建日期：2026-09-17  
> 状态：正式知识文档；依据培训材料重建，掌握程度待测试。  
> 唯一主线来源：`MMU.pdf` 的 **MMU S3 Introduction and Topology** 模块，PDF 第 58–68 页（课件页 3–23）。

## 结论先行

这一模块真正要建立的不是一张固定的 MMU S3 框图，而是一套分析任何集成方案的方法：

1. **SMMU 为什么存在**：把处理器 MMU 的地址翻译、权限检查和属性转换能力扩展到 DMA 等系统设备访问。
2. **MMU S3 为什么采用分布式结构**：TBU 靠近设备处理高频翻译查询，TCU 集中完成配置查询、页表遍历和系统级控制。
3. **如何区分拓扑**：分别追踪真实数据、翻译请求和维护消息经过哪里，以及翻译结果缓存在哪里。
4. **如何评价集成方式**：Inline、Cached、Lookaside 不是“谁更先进”，而是在数据路径、面积、时延、设备可信度和安全检查位置之间做不同取舍。

可以先用下面这张逻辑图记住全局关系：

```text
设备数据访问 ───────────────→ 数据路径（是否经过 SMMU 取决于集成方式）
      │
      └─ 翻译需求 → TBU / PCIe ATS requester
                         │ DTI message
                         ▼
                     BAS interconnect
                         │
                         ▼
                        TCU ──→ 配置表 / Translation Table / GPT
```

DTI/BAS 传递的是翻译与控制信息，不是设备 DMA payload。

---

## 1. 从处理器 MMU 到 System MMU

### 1.1 MMU 的三项基本职责

培训材料将 MMU 的功能归纳为：

- **Address translation**：通过 Translation Table Walk 把虚拟地址映射到物理地址，并用 TLB 缓存近期翻译；
- **Access permission control**：依据 Translation Table、Completer-side filtering 或 Granule Protection Check（GPC）限制访问；
- **Memory attribute conversion**：把翻译所得内存属性转换为系统互连和目标端能够执行的访问行为。

因此，MMU 不只是“VA 转 PA”。最终结果还会影响能否访问以及事务应采用什么属性。

### 1.2 SMMU 与处理器 MMU 的差别

System MMU 是放在设备下游、为系统设备请求提供翻译和保护的独立 MMU。它与处理器 MMU 的共同点是使用 VMSA Translation Table 格式；主要差别是服务对象和集成位置：

| 对比项 | 处理器 MMU | System MMU |
|---|---|---|
| 服务对象 | 处理器执行的 Load/Store/Instruction Fetch | DMA、PCIe、加速器等设备访问 |
| 上下文来源 | 当前处理器执行状态及系统寄存器 | 设备请求身份与 SMMU 的 Stream 配置 |
| 部署位置 | 处理器内部 | 设备与系统互连/内存系统之间或作为翻译服务 |
| 翻译能力 | 由处理器 Translation Regime 决定 | 可配置 Stage 1 only、Stage 2 only 或 S1+S2 |

SMMU 不负责替设备搬运数据。设备或 DMA engine 仍是请求发起者，SMMU负责解释地址、检查权限并形成正确的系统访问。

### 1.3 MMU S3 在系统中的位置

培训示例把 MMU S3 放在 PCIe、DMA、硬件加速器等 I/O Master 与 CMN/NOC/内存控制器之间。这个图表达的是系统角色，不是固定实例数：

- 一个系统可以有多个 TBU，为不同设备侧接口服务；
- 多个 TBU 可以共享一个 TCU；
- 不同设备可以在同一系统中选择不同集成方式；
- 具体 TBU 数量、端口宽度和 SID 路由由 SoC 集成决定。

---

## 2. MMU S3 的能力边界

培训材料把 MMU S3 描述为 SMMUv3 Architecture Specification 的实现，覆盖：

- SMMUv3.3；
- SMMU for RME；
- SMMU for RME DA；
- Stage 1 与 Stage 2 Translation；
- GPC、Realm Translation、MEC 等 CCA 相关能力；
- MMU S3 r1 起的 DPT 支持。

这里需要区分“产品系列架构方向”和“当前实例已实现能力”。培训后文明确称 DPT 是 future MMU S3 r1 feature，因此不能仅凭本页能力列表，就推断任意 MMU S3 版本或当前项目已经具备 DPT/RME DA。

### 2.1 Stage 1 与 Stage 2 的系统含义

```text
Stage 1：VA  → IPA
           由 OS 建立每个进程的地址空间

Stage 2：IPA → PA
           由 Hypervisor 为 Guest OS 建立隔离与重映射
```

仅启用 Stage 1、系统不存在 Hypervisor 时，培训把 IPA→PA 视为 flat mapping。应理解为没有额外的 Stage 2 重映射，而不是软件还必须执行一遍“恒等 Stage 2 walk”。

---

## 3. 分布式组件：TBU、TCU 与 BAS

### 3.1 TBU：靠近设备的翻译前端

Translation Buffer Unit（TBU）的核心是缓存和使用翻译信息。培训给出的 ACE-Lite TBU 由两部分构成：

- **BIU（Bus Interface Unit）**：位于设备真实数据通路中，接收并继续发出总线事务；
- **TLBU（Translation Lookaside Buffer Unit）**：保存 TLB，并向 BIU 提供翻译结果。

在 ACE-Lite TBU 内，BIU 与 TLBU 通过 LTI 紧耦合。TLBU 命中时本地返回；未命中时再通过 DTI/BAS 向 TCU 请求服务。

关键点：TBU 不等于一颗 TLB。ACE-Lite TBU 还包含承载数据事务的 BIU；而 LTI TBU 可以只有 TLBU，作为独立的翻译服务前端。

### 3.2 TCU：集中式翻译与控制中心

Translation Control Unit（TCU）负责需要访问系统内存和集中协调的工作：

- Configuration Table Walk（CTW）；
- Translation Table Walk / Page Table Walk（PTW）；
- Granule Protection Table Walk（GPT Walk）；
- SMMU Programming Interface。

培训把 TCU 分为：

- **TMU（Translation Management Unit）**：执行和管理翻译相关查询；
- **PIU（Programming Interface Unit）**：承载软件编程接口。

TCU 的集中化使多个设备侧 TBU 可以共享配置和页表查询能力，但不意味着每笔访问都到 TCU。TBU TLB 命中时可以在设备附近完成快速路径。

### 3.3 DTI 与 BAS：协议语义和物理承载

| 名称 | 层次 | 作用 |
|---|---|---|
| DTI | 消息协议 | 在 TBU、TCU、PCIe Root Port 之间表达翻译、失效、同步等消息 |
| BAS | 物理互连 | 在 MMU S3 中承载 DTI message，连接多个 TBU/PCIe Root Port 与一个 TCU |

BAS interconnect 可以由 BAS sizer、switch 和 register slice 组成，处理宽度适配、路由与时序切分。不能把“DTI over BAS”理解成 DTI 和 BAS 是两个串行完成相同工作的协议：前者定义消息含义，后者提供传输这些消息的物理通路。

### 3.4 LTI：紧耦合组件间的即时翻译接口

Local Translation Interface（LTI）是 channel-based interface，用于紧耦合组件之间发送翻译请求。培训强调其 response 必须立即使用，不能作为调用方可长期缓存的翻译结果。

这不等于“使用 LTI 的路径完全没有缓存”：

- LTI TBU 内部的 TLBU 仍然可以用 TLB 缓存翻译；
- 不能缓存的是通过 LTI 返回给请求方、供当前事务使用的 response；
- PCIe ATS 的 ATC 缓存属于另一种协议与组件边界。

---

## 4. 分析拓扑的统一方法

面对任意 MMU S3 拓扑，应分别回答以下问题：

1. **Data path**：真实 Read/Write payload 是否经过 SMMU 的 BIU？
2. **Translation request path**：由 BIU、LTI requester 还是 PCIe ATS requester 发起翻译请求？
3. **Translation cache location**：结果缓存在 TBU TLB、设备 ATC，还是不允许由请求方缓存？
4. **Miss path**：本地未命中后是否通过 DTI/BAS 到 TCU？
5. **Protection point**：Stage 2、GPC、DPT 或其他安全检查发生在哪里？

这五个问题比记住某张框图更可靠，因为三种集成方式可以在同一 SoC 中并存。

---

## 5. Inline Integration

Inline 是最直接的部署方式：设备所有事务进入 ACE-Lite TBU，设备不需要知道背后存在翻译。

```text
Device transaction
  → BIU
  → BIU 通过 LTI 请求 TLBU
      ├─ TLB hit：立即返回翻译
      └─ TLB miss：DTI-TBU → BAS → TCU → CTW/PTW/GPT
  → BIU 使用结果把事务发向 System interconnect
```

### 5.1 数据与控制路径

- **真实数据**：Device → BIU → System interconnect；
- **本地翻译请求**：BIU → LTI → TLBU；
- **远端 miss 请求**：TLBU → DTI-TBU/BAS → TCU；
- **页表/GPT 访问**：TCU → System interconnect → Memory。

Inline 的优点是设备透明、保护位置集中；代价是所有数据事务都经过 SMMU 数据通路，BIU 的带宽、缓冲和时延成为集成设计的一部分。

---

## 6. Cached Integration：PCIe ATS 与 ATC

Cached Integration 在 Inline 数据通路的基础上加入 PCIe ATS：设备可以提前请求翻译，并把结果保存在设备侧 Address Translation Cache（ATC）中。

```text
ATS Translation Request
  → PCIe controller
  → DTI-ATS / BAS
  → TCU
  → Translation Completion
  → Device ATC

后续数据事务仍经过 ACE-Lite TBU/BIU
```

“Cached” 的关键不是数据绕过 SMMU，而是设备提前缓存翻译。培训明确指出所有事务仍经过 SMMU，只是已翻译阶段不必重复完成。

### 6.1 Full ATS

- ATC 缓存 Stage 1 + Stage 2 的完整翻译结果；
- 设备发出 translated transaction；
- SMMU 不再重做这两个翻译阶段，但事务仍经过数据路径并接受适用的安全检查。

优势是设备能直接使用最终翻译，减少在线翻译延迟；风险是完整翻译结果交给设备后，若设备不可信，系统必须依靠后续安全检查阻止其伪造 translated address。

### 6.2 Split-stage ATS

- ATC 只缓存 Stage 1 结果；
- 设备发出的地址仍需由 SMMU 在线完成 Stage 2；
- Stage 2 保留在受系统控制的数据路径中，对不可信设备提供更强隔离。

代价是 translated transaction 仍要经过 Stage 2；培训还指出它对 fully coherent device 和 PCIe peer-to-peer ACS 流量存在功能限制。

---

## 7. DPT：在 Full ATS 灵活性上补充权限检查

培训用 Full ATS 与 Split-stage ATS 的缺点引出 Device Permission Table（DPT）：

| 方案 | 主要优点 | 培训指出的不足 |
|---|---|---|
| Full ATS | 缓存完整 S1+S2 翻译，功能灵活 | GPC 可保护 PAS，Realm 可防 rogue Hypervisor，但不同 Realm 之间仍缺少足够隔离 |
| Split-stage ATS | Stage 2 留在 SMMU，设备更难绕过隔离 | fully coherent、P2P ACS 等场景受限 |
| DPT | translated transaction 不必重做完整 Stage 2，但仍接受目标权限检查 | 属于 MMU S3 r1 的后续能力，需具体实现支持 |

DPT 是位于内存中的查找表，按物理内存区域记录允许的 VMID。收到 translated transaction 时，DPT check 检查该 StreamID 所代表的访问上下文是否允许到达目标区域；DPT entry 也可以被缓存。

因此，DPT 的定位不是第三阶段地址翻译，而是对已经翻译到目标 PA 的设备事务增加细粒度许可检查。

---

## 8. Lookaside Integration：Translation-as-a-Service

Lookaside 把 MMU S3 放在翻译服务路径旁边，而不是放在真实数据路径中：

```text
Controller 请求翻译
  → LTI TBU / DTI-ATS
  → DTI/BAS
  → TCU
  → 返回翻译信息

Controller 使用翻译结果
  → 直接向 System interconnect 发起数据事务
```

培训图中的核心特征是：

- 设备/控制器向 SMMU 请求 translation；
- SMMU 返回 translation information；
- 真实事务不经过 SMMU；
- PCIe controller 可以直接连接 system interconnect；
- controller 自己维护 ordering rule；
- SMMU 不需要为该数据路径提供 write data buffer，可节省面积；
- PCIe peer-to-peer ACS 流量不必为了翻译而进入主互连。

Lookaside 并不等于没有保护。它把关键责任前移到“翻译服务返回什么”和“控制器如何使用结果”，并要求系统集成保证未经授权的直达事务不能绕过应有检查。

---

## 9. 三种集成方式的准确对比

| 维度 | Inline | Cached | Lookaside |
|---|---|---|---|
| 数据是否经过 SMMU | 是 | 是 | 否 |
| 设备是否提前请求翻译 | 否，访问到达后由 TBU 处理 | 是，PCIe ATS | 是，Translation-as-a-Service |
| 设备侧是否缓存翻译 | 否 | 是，ATC | LTI response 立即使用；其他协议缓存能力另行判断 |
| SMMU 数据路径组件 | ACE-Lite TBU 的 BIU + TLBU | ACE-Lite TBU；另有 DTI-ATS 到 TCU | LTI TBU 可只含 TLBU，不承担数据 payload |
| 主要优点 | 设备透明、保护集中 | 降低设备访问的在线翻译时延 | 数据不经过 SMMU，节省数据缓冲和路径面积 |
| 主要集成责任 | SMMU 承担全部数据路径性能 | ATS/ATC 一致性及 Full/Split 安全选择 | Controller ordering、直达路径和保护边界 |

最容易混淆的是 Cached 与 Lookaside：二者都可能提前取得翻译，但 Cached 的数据仍穿过 SMMU，Lookaside 的数据直接进入系统互连。

---

## 10. PCIe Bifurcation 示例在说明什么

Bifurcation 把一个较宽的 PCIe 资源拆分为多个逻辑控制器/端口。培训用 x16、x8、x4、x4 组合说明：PCIe 拆分后，MMU S3 的数据接口和翻译接口也必须按新的请求来源进行集成。

### 10.1 ACE-Lite Bifurcation

示例中多个 PCIe controller 的数据访问先汇聚到 NOC S3，再通过 Dual TBU 接入 CMN S3；翻译控制还通过 DTI-TBU、DTI-ATS、BAS 和 TCU 完成。

学习重点不是记住“必须用两个 ACE-Lite TBU”，而是：

- 分叉后的多个数据源可以在 NOC 侧汇聚；
- 共享/并行 TBU 必须满足总带宽和独立请求上下文；
- ATS 控制路径与数据通路分开规划；
- SID 和响应路由必须能区分各逻辑控制器。

### 10.2 LTI Bifurcation

示例把四个 LTI channel 接入一个 LTI x4 TBU：

- controller 的真实数据通过 NOC S3 直接进入 CMN S3；
- translation request 通过各自 LTI channel 进入 LTI TBU；
- LTI TBU 未命中时再通过 DTI-TBU/BAS 到 TCU；
- PCIe ATS 所需的 DTI-ATS 路径仍可独立存在。

因此，LTI x4 表示多个紧耦合 translation channel 的聚合能力，不表示四路 DMA payload 经过 LTI TBU。

---

## 11. 一条端到端主线

```text
设备发起访问
  → 根据集成方式决定：数据进入 ACE-Lite TBU，还是由 Controller 直达互连
  → 翻译需求在 TBU TLB、设备 ATC 或 LTI translation service 中处理
  → 本地未命中时，DTI message 经 BAS 到 TCU
  → TCU 获取 Stream/Context 配置，并执行所需 PTW/GPT walk
  → 翻译结果返回请求方
  → 在适用位置执行 Stage 2、GPC、DPT 等保护检查
  → 数据事务按所得地址、权限和属性继续进入系统
```

## 12. 培训材料的使用边界

1. **模块边界**：本篇只使用 PDF 第 58–68 页，不再混入旧文档中的第 32–33、74 页或历史问答推导。
2. **示意图不是实例表**：图中的 TBU、Root Port、NOC 数量不能直接套到当前 SoC。
3. **DPT 是版本能力**：培训明确标注为 MMU S3 r1 future feature；使用前必须核对当前产品版本。
4. **“Security checks”不是固定的一种检查**：Full ATS 图只说明 translated transaction 仍接受适用安全检查，不能据此断言一定重做 Stage 2。
5. **LTI response 不可缓存有明确边界**：限制的是调用方保存该 response；不能由此推断 LTI TBU 内没有 TLB。
6. **Bifurcation 图讲集成原则**：具体 lane 划分、SID 生成、路由、带宽和排序规则仍由系统集成定义。

## 面试版总结

> MMU S3 采用分布式 SMMU 结构：TBU 靠近设备缓存并使用翻译，TCU 集中完成配置、页表和 GPT 查询，DTI 消息通过 BAS 在两者之间传递。Inline、Cached、Lookaside 的根本区别在于真实数据是否经过 SMMU、翻译结果缓存在何处，以及最终安全检查放在哪里。

## 来源

- [MMU.pdf：MMU S3 Introduction and Topology](../../99_SOURCE/Training/MMU.pdf)，PDF 第 58–68 页。

