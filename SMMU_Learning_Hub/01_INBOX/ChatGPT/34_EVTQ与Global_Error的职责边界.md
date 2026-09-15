# EVTQ 与 Global Error 的职责边界

记录日期：2026-09-02  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，培训材料截图  
状态：Inbox；寄存器字段、Event 合并条件和错误恢复细节待结合 SMMUv3/MMU-720AE 文档校验

## 问题

SMMUv3 的 Event Queue 如何工作？它与 Command Queue、Global Error 分别承担什么职责？发生队列溢出、CMDQ 错误、EVTQ 访问异常或 MSI 写失败时，软件如何得到通知并恢复？

## 回答

### 核心结论

> **EVTQ 主要报告“某笔 transaction 或某个 Stream 发生了什么”；Global Error 主要报告“SMMU 自身的控制、队列或通知基础设施出了什么问题”。**

二者不是互斥关系：对于 CMDQ error，Global Error 可以提供全局告警和停机状态，EVTQ record 则可以补充具体错误原因。

## 1. EVTQ 是 SMMU 到软件的异步报告通道

CMDQ 与 EVTQ 的通信方向相反：

```text
CMDQ：Software → SMMU
      “请执行这些管理命令”

EVTQ：SMMU → Software
      “刚才发生了这些事件或错误”
```

因此两种队列的 Producer/Consumer 也相反：

| 队列 | Producer | Consumer |
| --- | --- | --- |
| CMDQ | Software | SMMU |
| EVTQ | SMMU | Software |

EVTQ 的概念流程是：

```text
SMMU 产生 Event
       ↓
写入内存中的 EVTQ，并推进 PROD
       ↓
按配置产生中断
       ↓
Software 读取并处理 Event Record
       ↓
Software 推进 CONS
```

`EVTQ_BASE` 描述队列在内存中的位置和大小；SMMU 推进 `EVTQ_PROD`，软件推进 `EVTQ_CONS`。

## 2. EVTQ 报告的典型内容

EVTQ 用于容纳大量、异步、多 Stream 的 event/fault record，典型内容包括：

- 非法或无法识别的 Command，例如 `CERROR_ILL`。
- 超出实现范围的 StreamID，例如 `C_BAD_STREAMID`。
- 无效或配置非法的 Context Descriptor，例如 `C_BAD_CD`。
- 与 transaction、translation、configuration 或 Stream 相关的其他事件。

相较单一 fault register，队列能够连续保存多个事件，降低新错误覆盖旧错误的风险，并携带更多诊断上下文。

## 3. Overflow 与 Event Merging

如果 SMMU 产生 event 的速度超过软件消费速度，PROD 追上 CONS，EVTQ 就可能溢出，并导致 event record 丢失。因此软件通常借助中断及时消费队列，而不是仅靠低频轮询。

为减轻同一 Stream 连续产生大量相似事件造成的队列压力，架构允许在规定条件下合并 event。若软件需要更细粒度的独立记录，可根据 STE 中与 merging 相关的控制（如 `STE.MEV`）进行配置；准确适用范围仍应以规范字段定义为准。

## 4. EVTQ 与 Stall Fault 的处理闭环

当某个 Stream 采用 Stall 模式并发生可恢复 fault 时，可以形成如下闭环：

```text
Device transaction
       ↓
SMMU 发现 fault 并暂挂请求
       ↓
EVTQ 把 SID、地址和原因报告给 Software
       ↓
Software 修复页表或配置
       ↓
Software 通过 CMDQ 提交 Resume 等命令
       ↓
SMMU 继续或终止原请求
```

因此可以概括为：

> **EVTQ 把问题告诉软件，CMDQ 把软件的处理决定送回 SMMU。**

## 5. Global Error 处理的是管理与基础设施错误

Global Error 更关注影响 SMMU 全局运行或编程接口的错误，例如：

- Command Queue error。
- SMMU 访问内存中的 Event Queue 时发生 external abort。
- SMMU 为发送中断而执行 MSI write 时发生 abort。

这类错误由 `SMMU_(S_)GERROR` 一类状态寄存器报告，并可按配置触发 Global Error interrupt。

## 6. 为什么有些错误不能只依赖 EVTQ

如果 SMMU 写 EVTQ memory 本身发生 abort，那么错误记录已经无法可靠写入 EVTQ；此时必须使用独立的 Global Error 路径报告。

MSI write abort 也具有相同性质：SMMU 的通知通道本身失败了。若 Global Error interrupt 仍依赖同一个已经失败的 MSI target，就会出现“用坏掉的通道报告该通道已经损坏”的问题，因此系统集成必须考虑可靠的错误通知路径。

## 7. CMDQ Error 为什么同时涉及 GERROR 与 EVTQ

CMDQ 是软件控制 SMMU 状态的管理通道。命令流非法或队列出错后，继续执行后续命令可能进一步破坏 configuration/translation 状态，因此 SMMU 会停止继续处理相关后续命令，等待软件处理并清除错误。

此时二者可以分工：

```text
GERROR
→ 表示发生了全局 CMDQ 错误，并反映控制路径被暂停

EVTQ record
→ 在能够正常写入 EVTQ 时，提供更具体的 command error 原因
```

软件应先读取状态和诊断记录、处理根因，再按规范通过 `SMMU_(S_)GERRORN` 的对应位确认/清除错误，使相关处理恢复。不能只清状态位而忽略根因。

## 面试版总结

> **Event Queue 是 SMMU 向软件异步报告 transaction、Stream、translation 和 configuration event 的内存环形队列，SMMU 是 Producer，软件是 Consumer。Global Error 则报告影响 SMMU 自身控制和通知路径的错误，例如 CMDQ error、EVTQ memory access abort 和 MSI write abort。CMDQ error 发生时，GERROR 提供全局告警与暂停状态，EVTQ 可提供具体原因；软件修复根因并清除错误后，相关处理才能恢复。**

## 一句话总结

> **CMDQ 是软件给 SMMU 下命令，EVTQ 是 SMMU 向软件报事件，GERROR 则在 SMMU 的管理或报告通道自身出问题时拉响全局警报。**

## 关联记录

- [SMMUv3 Programming Model](23_SMMUv3_Programming_Model.md)
- [CMDQ、PROD/CONS 与 CMD_SYNC](30_CMDQ_PROD_CONS与CMD_SYNC.md)
- [CMD_SYNC 的本质与完成边界](33_CMD_SYNC的本质与完成边界.md)
- [待研究问题：Secure MSI 与 ITS 路径](../Questions/2026-09-02_Secure_MSI与ITS路径.md)

