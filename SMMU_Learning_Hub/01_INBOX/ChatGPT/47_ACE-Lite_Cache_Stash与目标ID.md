# ACE-Lite Cache Stash 与目标 Node ID / LP ID

记录日期：2026-09-06  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，Training slide 中 ACE-Lite TBU interface properties 的 Cache Stash Transactions 部分（培训视频截图，时间点未知）  
状态：Inbox；`AWSNOOP` 精确编码、目标 ID 拓扑和 MMU-720AE TBU 的信号处理规则待结合 ACE5/ACE5-Lite 与产品 TRM 校验

## 问题

1. 什么是基于 ACE-Lite 的 Cache Stash Transaction？它解决什么问题，与普通 coherent DMA、prefetch 和 SMMU/TBU 分别有什么关系？
2. `AWSTASHNID` 和 `AWSTASHLPID` 应该如何理解，这些 ID 由谁定义、又由谁设置？

## 回答

### 核心结论

> **Cache Stash 是 Device→CPU producer-consumer 场景下的 cache locality 优化：Device 在产生或写入数据时携带 stash 信息，请求 coherent system 把相关 cache line 放到目标处理单元附近。SMMU/TBU 负责地址翻译并正确处理这些 sideband，coherent interconnect 才负责解释目标并执行 cache placement。**

## 1. Cache Stash 解决什么问题

以 NIC 接收数据包为例。没有 Cache Stash 时：

```text
NIC DMA write → LLC / Memory
                      ↓
CPU 随后读取 → Cache miss → 从较远层级取回
```

如果 Device 知道 CPU 很快就会消费这批数据，就可以在 DMA transaction 中表达一个 stash intent：

```text
NIC DMA write + Stash target
              ↓
       Coherent Interconnect
          ├─ 维护正确的内存/一致性状态
          └─ 将 cache line 放到目标 CPU/Node 附近
```

这样，目标 CPU 随后 load 时更可能在较近的 cache hierarchy 中命中，从而降低访问延迟和互连流量。

因此，Cache Stash 本质上是：

> **producer 已知 consumer 时，对数据进行定向的 cache placement/pre-positioning。**

## 2. 它不是 Device 直接写 CPU L1 Cache

Device 仍然只向 coherent interconnect 发起协议 transaction，并不直接寻址某个 CPU 的 L1/L2 Cache。

最终落在哪一级 Cache、以何种 cache state 保存，以及目标无法接收时如何处理，取决于：

- coherent topology；
- interconnect 和 cache implementation；
- stash transaction type；
- 目标节点支持能力和当前状态。

因此更准确的说法是“向系统提出 cache placement 请求或提示”，而不是保证“数据必然写入某个指定 L1 Cache”。

## 3. 与普通 coherent DMA、prefetch 的区别

| 机制 | 首要目标 | 关键特点 |
| --- | --- | --- |
| Coherent DMA | Correctness / Coherency | 保证 Device 写入后 CPU 能看到正确数据 |
| Cache Stash | Performance / Locality | 在保持一致性的同时，把数据往已知 consumer 的 cache hierarchy 放置 |
| 普通 Prefetch | 提前获取将来可能使用的数据 | 通常由处理器或硬件预测未来访问，不一定知道明确 consumer |

一句话区分：

> **Coherent DMA 是“把数据写对”，Cache Stash 是“顺便把数据送到马上要用它的处理单元附近”。**

## 4. ACE-Lite 中的相关信号

Training slide 给出了下列写地址通道相关字段：

| 信号 | 作用 |
| --- | --- |
| `AWSNOOP[3]` | 参与编码 Cache Stash transaction 类型；精确编码以 ACE 规范为准 |
| `AWSTASHNID` | 目标 physical coherent interface 的 Node ID |
| `AWSTASHNIDEN` | 表示 `AWSTASHNID` 是否有效 |
| `AWSTASHLPID` | 该 physical interface 后面的 logical processor/subunit ID |
| `AWSTASHLPIDEN` | 表示 `AWSTASHLPID` 是否有效 |

这些字段表达的是 stash 语义和目标，不是地址翻译上下文。SID/SSID、地址和 stash target 分别解决不同问题。

## 5. Node ID 与 LP ID 为什么分两级

系统中多个 CPU Core 或逻辑单元可能共享一个对 coherent fabric 的物理接口：

```text
Coherent Interconnect
        ↓
Physical Interface / DSU
   ├─ Logical Processor 0
   ├─ Logical Processor 1
   ├─ Logical Processor 2
   └─ Logical Processor 3
```

因此需要两级定位：

```text
Node ID → 先找到哪个 physical coherent interface
LP ID   → 再找到该接口后面的哪个 logical processor/subunit
```

最容易记的一句话是：

> **NID 找“哪个门”，LPID 找“进门以后哪个人”。**

但不要把它机械等同为：

```text
Node ID = Cluster ID
LP ID   = CPU Core ID
```

在某些 SoC 上可能近似如此，但协议定义的是 physical interface 和 associated logical processor/subunit，具体映射属于系统实现。

## 6. Enable 组合表达的目标精度

| `NIDEN` | `LPIDEN` | 概念含义 |
| ---: | ---: | --- |
| 0 | 0 | 不提供具体 NID/LPID 目标 |
| 1 | 0 | 只指定 physical interface |
| 1 | 1 | 同时指定 physical interface 与其后的 logical processor |
| 0 | 1 | 协议定义的特殊使用情形，不能按普通两级目标模型随意使用 |

解释任何 ID 之前必须先检查相应 Enable；总线上存在一个数值，不代表该字段在当前 transaction 中有效。

## 7. 这些 ID 是怎么设置的

ACE 协议定义字段的语义，但不统一规定“CPU0 的 NID 必须是多少”。具体编码通常由下列因素共同决定：

```text
SoC coherent topology
+ interconnect configuration
+ CPU / DSU / accelerator integration
```

真正发起 stash transaction 的 master 最终在 AW channel 上驱动 `AWSTASH*` 信号。它获得目标值通常有两种方式：

1. **软件配置**：OS/Driver 根据 queue affinity 或工作负载分配，把目标 NID/LPID 写入 Device 寄存器。
2. **固定硬件映射**：某些 accelerator 的 consumer 固定，目标由 RTL 或系统集成静态确定。

到底是软件可编程还是固定映射，属于具体 Device/IP 和 SoC 的实现事实。

## 8. NIC Queue → CPU 的完整例子

假设系统集成定义：

```text
DSU0 physical interface → NID 0x10
DSU0 后的 CPU2          → LPID 2
```

软件把 NIC RX Queue 3 分配给 CPU2，于是配置 Device：

```text
RXQ3_STASH_TARGET:
    NID  = 0x10
    LPID = 2
```

NIC 收到 Queue 3 的 packet 后发起：

```text
DMA Write
+ AWSNOOP = Stash transaction type
+ AWSTASHNIDEN  = 1
+ AWSTASHNID    = 0x10
+ AWSTASHLPIDEN = 1
+ AWSTASHLPID   = 2
```

其中 `0x10` 和 `2` 只是示例；实际数值必须来自当前 SoC 的 coherent topology 与 Device 配置接口。

## 9. Cache Stash 穿过 SMMU/TBU 时的职责分工

```text
Device
  │ IOVA + SID/SSID + AWSNOOP + STASH NID/LPID
  ▼
TBS → TBU → TBM
       │
       ├─ 根据 SID/SSID 选择 translation context
       ├─ IOVA → PA
       └─ 正确处理/传递 stash attributes
  ▼
Coherent Interconnect
       │
       └─ 解释 NID/LPID，并执行支持的 stash 行为
```

要点是：

- **SMMU/TBU** 决定地址如何翻译、访问是否允许，并保证 stash sideband 与输出 transaction 正确对应。
- **Coherent interconnect/cache system** 解释 stash destination，执行具体 cache placement。
- **Device/软件或固定集成逻辑** 决定并提供目标 NID/LPID。

TBU 不是根据地址计算 NID/LPID，也不负责定义“哪个 NID 对应哪个 CPU”。

## 10. 证据边界

- **Training slide 明确内容**：ACE-Lite TBU interface 支持 Cache Stash Transactions，并列出 `AWSNOOP[3]`、NID/LPID 与 Enable 字段。
- **ACE 架构背景**：NID 标识目标 physical interface，LPID 标识与该接口关联的 logical processor/subunit。
- **系统集成事实**：具体 ID 数值、软件寄存器、物理 cache level、stash 完成行为和 TBU 信号变换必须查 SoC、Interconnect、Device 和 MMU-720AE 文档。

## 一句话总结

> **ACE-Lite Cache Stash 让 I/O Device 在 DMA 时携带目标信息，请求 coherent system 把数据预先放到未来 consumer 附近；NID 选择 physical coherent interface，LPID 进一步选择其后的 logical processor，SMMU 只负责地址翻译和 sideband 处理，目标编码与实际 cache placement 由 SoC 集成和 coherent system 决定。**

## 相关记录

- [TBU、TCU、DTI 与 DTI-ATS](01_TBU_TCU_DTI与DTI-ATS概述.md)
- [DMA 机制、SoC 应用与 SMMU 隔离](45_DMA机制_SoC应用与SMMU隔离.md)
