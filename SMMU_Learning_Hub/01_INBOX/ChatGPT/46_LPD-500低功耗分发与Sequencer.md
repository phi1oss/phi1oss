# LPD-500 低功耗分发与 Sequencer 行为

记录日期：2026-09-04  
记录端：手机 / GitHub；本地整理于 2026-09-06  
来源：[GitHub 手机端原记录](https://github.com/phi1oss/phi1oss/blob/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93/00_Inbox/2026-09-04_1719_LPD-500%E4%BD%8E%E5%8A%9F%E8%80%97%E5%88%86%E5%8F%91%E4%B8%8ESequencer%E8%A1%8C%E4%B8%BA.md)  
状态：Inbox；Q-Channel 的精确信号时序、LPD-500 配置方式和 MMU-720AE FuSa 行为待结合相应 TRM 核实

## 问题

LPD-500 在 MMU-720AE 低功耗控制中的作用是什么？Expander 与 Sequencer 有何区别？Sequencer 是否可以只关闭组内某一个 TBU？

## 回答

### 核心结论

> **LPD-500 把一个上游低功耗请求协调到一组下游组件：Expander 让成员并行响应，Sequencer 让成员按配置顺序响应。Sequencer 改变的是顺序，不是从同一组里动态挑选某一个组件；真正独立的电源控制通常需要独立分组或独立控制路径。**

### 1. LPD-500 做什么、不做什么

LPD（Low Power Distributor）位于上游 power/clock controller 与下游组件之间，通过 Q-Channel 类握手协调低功耗状态转换。下游组件可以是 TBU、TCU 或其他需要先 quiesce 的单元。

```text
Power/Clock Controller
          ↓  low-power request
        LPD-500
     ↙      ↓      ↘
   TBU0    TBU1     TCU
```

LPD-500 的核心工作是：

- 向下游分发低功耗请求。
- 收集组件是否接受、拒绝或仍处于活动状态的反馈。
- 只有在整组满足条件后，才向上游表示可以继续状态转换。

它通常不直接驱动物理 power switch 或 clock gate；真正切断电源/时钟的是上游电源控制基础设施。Q-Channel 完成表示组件已经达到允许状态转换的条件，不等于“此刻已经物理断电”。

### 2. 一个上游请求控制的是一个下游组

如果多个组件接在同一个 LPD 低功耗控制组内，上游发来的是针对该组的请求。LPD 不会在每次请求中临时解析“这次只选 TBU0，下次只选 TCU”。

若 TBU0、TBU1 和 TCU 必须被软件或硬件真正独立地控制，通常需要：

- 不同的 low-power group。
- 独立的 LPD/Q-Channel path。
- 或产品明确提供的独立控制机制。

精确能力必须依据 LPD-500 和 SoC power-domain integration 的寄存器及连接图确认。

### 3. Expander 与 Sequencer

| 模式 | 行为 | 适合场景 |
| --- | --- | --- |
| Expander | 把同一个请求并行分发给组内成员，并汇总反馈 | 成员之间没有严格先后依赖，希望降低进入/退出延迟 |
| Sequencer | 按配置顺序把同一个组请求逐步传递给成员 | 组件存在依赖，必须先让边缘 requester 停止，再处理中央资源 |

一个概念性的关停顺序可以是：

```text
TBU0 quiesce → TBU1 quiesce → TCU quiesce → group accepted
```

恢复时是否采用相反顺序以及每一步的精确信号条件，应以 LPD-500 配置和 TRM 时序为准。

### 4. Sequencer 为什么不能理解为“只关一个”

Sequencer 的目标仍是让整个参与组完成一次低功耗握手，只是把“同时做”改成“按顺序做”。

- 某个成员未 ready 或返回 deny，可能阻止整组请求完成。
- 某个成员完成握手，不表示其余成员可以永远跳过。
- 最终是否真正关电，由上游控制器在整组满足条件后决定。

因此，**顺序控制**与**独立选择控制**是两个不同问题。

### 5. 与 MMU-720AE 和 FuSa 的关系

在 MMU-720AE 集成中，低功耗切换必须避免 TBU/TCU 尚有未完成 transaction、translation 或维护操作时直接关断。Sequencer 可以用来表达组件之间的依赖顺序。

带 FuSa 能力的实现还可能对 Q-Channel 的异常状态、非法时序或握手故障进行检测，并通过相应安全错误通路报告给 TCU FMU。具体覆盖范围和错误响应属于产品实现事实，必须查 MMU-720AE/LPD-500 TRM，不能仅由通用 Q-Channel 概念推断。

## 一句话总结

> **LPD-500 协调一个低功耗组安全进入或退出目标状态：Expander 并行分发，Sequencer 按序分发；若要单独控制某个 TBU，需要独立分组或产品明确支持的独立路径。**
