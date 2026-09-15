# SMMU PMCG 与 PMU Snapshot

记录日期：2026-09-07  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，Training slide 中 MMU S3/SMMUv3 Performance Monitor 与 Snapshot 接口（培训视频截图，时间点未知）  
状态：Inbox；PMCG 数量、事件编号、StreamID filter、snapshot 寄存器和握手时序待结合 MMU-720AE TRM 核实

## 问题

如何理解 MMU S3/SMMUv3 的 PMCG、实时性能计数器和 PMU Snapshot？为什么每个 TBU/TCU 都有独立 PMCG，`pmusnapshot_req/ack`、`SMMU_PMCG_SVRn` 与 CoreSight CTI 又分别起什么作用？

## 回答

### 核心结论

> **PMCG 是 TBU/TCU 的性能计数器组，用于统计 translation、cache 和内部实现事件；实时计数器持续变化，而 Snapshot 会在一个确定时刻把多个 counter 的值复制到 shadow registers，使软件能够读取同一时间截面的性能状态。**

## 1. PMCG 是什么

PMCG 是 **Performance Monitor Counter Group**。在 Training 所描述的 MMU S3/MMU-720AE 结构中，TBU 和 TCU 可分别具有独立 PMCG：

```text
TBU0 → PMCG0
TBU1 → PMCG1
TCU  → PMCG2
```

PMCG 可以通过 `SMMU_PMCG_*` 寄存器配置和读取硬件事件，例如：

- translation request/response 活动；
- TLB hit/miss；
- configuration 或 translation walk 相关事件；
- TBU/TCU 的实现定义 micro-architecture 事件；
- 在相应过滤能力启用时，特定 StreamID 的活动。

它主要用于性能分析、瓶颈定位、测试和调试，不参与地址翻译的功能正确性决策。

## 2. 为什么 TBU 与 TCU 分开统计

TBU 与 TCU 的性能职责不同：

```text
TBU
├─ inline transaction processing
├─ local translation/cache lookup
└─ DTI translation request

TCU
├─ configuration lookup
├─ translation table walk
├─ backend translation/configuration cache
└─ centralized control
```

如果所有事件混在一套 counter 中，就难以判断：

- 哪个 TBU 出现高 miss rate；
- 性能瓶颈在本地 lookup、DTI 还是 TCU backend；
- 某个 Device/Stream 是否造成异常 translation 压力。

独立 PMCG 能把不同实例和处理阶段分开观察。若 PMCG 支持 StreamID filter，还可以只统计某个 requester，例如 `SID=20` 的 TLB miss；具体 filter 字段和适用事件需要查 TRM。

## 3. 实时读取 Counter

软件可配置某个 event counter 统计特定事件，再读取 `SMMU_PMCG_EVCNTRn`：

```text
EVTYPER0 = 某个 TLB-miss event
EVCNTR0  = 12345
```

其含义是从计数启用或上次清零以来，该事件累计发生了 12345 次。

实时 counter 在软件依次读取多个寄存器时仍可能继续增长：

```text
t0: read Counter0
t1: new event occurs
t2: read Counter1
t3: read Counter2
```

这样得到的 Counter0、Counter1、Counter2 并不严格属于同一个时刻。

## 4. PMU Snapshot 解决什么问题

Snapshot 在一个 capture point 把多个 live counter 的值复制到对应 shadow registers：

```text
                     Capture at time T
Live EVCNTR0 = 100 ────────────────> SVR0 = 100
Live EVCNTR1 = 200 ────────────────> SVR1 = 200
Live EVCNTR2 = 300 ────────────────> SVR2 = 300
```

Capture 完成后，live counters 可以继续计数：

```text
EVCNTR0: 100 → 130
SVR0:    100      // 保留 time T 的快照值
```

所以 Snapshot 更准确地说是“同时采样并复制”，而不是永久冻结 live counter。软件之后可以慢慢读取多个 `SMMU_PMCG_SVRn`，仍获得一致的时间截面。

> **Snapshot 就像给全部 PMU Counter 在同一瞬间拍一张性能照片。**

## 5. `pmusnapshot_req/ack` 四阶段握手

硬件 snapshot interface 采用 request/acknowledge 四阶段握手：

```text
Requester                         PMCG

req = 1  ────────────────────────>
                                  capture counters
         <─────────────────────── ack = 1

req = 0  ────────────────────────>
         <─────────────────────── ack = 0
```

四个阶段是：

1. Requester 拉高 `pmusnapshot_req`。
2. PMCG 捕获 counter 并拉高 `pmusnapshot_ack`。
3. Requester 看到 ack 后拉低 req。
4. PMCG 看到 req 解除后拉低 ack，接口回到 idle。

相比单周期 pulse，保持 req 直到收到 ack 更适合跨时钟或异步边界，可避免 capture 请求因为脉冲过窄而丢失。精确同步结构和时序约束属于产品实现。

## 6. Snapshot Value Registers

`SMMU_PMCG_SVRn` 是 Snapshot Value/Shadow Registers，用于保存 capture 瞬间对应 counter 的值：

| 寄存器 | 作用 |
| --- | --- |
| `SMMU_PMCG_EVCNTRn` | 持续变化的 live event counter |
| `SMMU_PMCG_SVRn` | 最近一次 snapshot 捕获的稳定值 |

Shadow register 让软件不必停止整个 PMU，就可以读取多项属于同一 snapshot 时刻的数据。

## 7. CoreSight CTI 为什么要触发 Snapshot

CoreSight Cross Trigger Interface（CTI）可以把一次系统调试触发分发给多个组件：

```text
CPU/Trace subsystem detects trigger
                 ↓
                CTI
       ┌─────────┼──────────┐
       ↓         ↓          ↓
CPU trace    SMMU PMU    Other IP
capture      snapshot    snapshot/trace
```

这样可以关联同一时间点的：

- CPU trace；
- SMMU translation/cache 活动；
- 其他互连或系统 IP 的状态。

它适合定位“CPU 出现延迟或异常时，SMMU 正在发生什么”。CTI 负责同步触发，PMCG 负责实际捕获 counter。

## 8. 软件触发与硬件触发

Snapshot 可以有两种入口：

```text
Hardware trigger:
CTI / debug logic → pmusnapshot_req

Software trigger:
write SMMU_PMCG_CFGR.CAPTURE
```

软件触发适合受控性能测量，例如在 workload 前后采样；硬件触发适合故障、trace 或跨 IP 同步分析。

具体产品是否支持两种方式、`CAPTURE` 字段写入语义以及多个触发同时到达时如何处理，应以 MMU-720AE TRM 为准。

## 9. 一个典型性能分析流程

```text
选择事件：TBU TLB miss、TCU table walk 等
                  ↓
配置 PMCG event type / filter
                  ↓
运行目标 Device workload
                  ↓
在关键时刻触发 snapshot
                  ↓
读取各 TBU/TCU 的 SVRn
                  ↓
比较 local miss、DTI request 与 backend walk
                  ↓
定位瓶颈所在实例和处理阶段
```

PMCG counter 表示事件发生次数；要形成 miss rate、平均延迟或带宽等派生指标，通常还需要选择分母事件、时间窗口并理解 event 的精确定义。

## 10. 证据边界

- **Training slide 明确内容**：每个 TBU/TCU 的独立 PMCG、snapshot request/ack、SVRn shadow register，以及 CoreSight CTI 和软件 CAPTURE 入口。
- **通用架构背景**：实时 counter 顺序读取会产生时间偏差，snapshot 用于获得统一时间截面；四阶段握手避免事件丢失。
- **待产品核实**：实例数量、counter 数量、事件编码、StreamID filter、溢出行为、寄存器字段和握手时序。

## 一句话总结

> **PMCG 是每个 TBU/TCU 的独立性能计数器组；`EVCNTRn` 提供实时累计值，Snapshot 通过硬件握手或软件 CAPTURE 把同一时刻的 counter 值复制到 `SVRn`，便于 CoreSight 和软件对多个组件进行时间一致的性能分析。**

## 相关记录

- [TBU、TCU、DTI 与 DTI-ATS](01_TBU_TCU_DTI与DTI-ATS概述.md)
- [CMD_SYNC 的本质与完成边界](33_CMD_SYNC的本质与完成边界.md)
