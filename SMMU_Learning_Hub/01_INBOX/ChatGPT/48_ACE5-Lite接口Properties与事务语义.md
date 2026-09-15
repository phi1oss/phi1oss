# ACE5-Lite 接口 Properties：能力声明与事务语义

记录日期：2026-09-06  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，Training slide 中 MMU-720AE ACE-Lite TBU interface properties 及 Loopback 追问（培训视频截图，时间点未知）  
状态：Inbox；各 Property 的精确约束、信号组合和 TBU 支持方式待结合 ACE5/AXI5 与 MMU-720AE TRM 校验

## 问题

1. ACE5-Lite TBU interface 表中的 `Property` 应该如何理解？它与每笔 transaction 携带的 attribute 或 signal 有什么区别？
2. `DeAllocation_Transactions`、`Atomic_Transactions`、`Loopback_Signals`、`Poison`、`Unique_ID_Support`、`Read_Data_Chunking`、`CMO_On_Read`、`Persist_CMO` 和 `Wakeup_Signals` 分别解决什么问题？
3. 为什么同一个 Master 可以发出多笔相同 AXI ID 的 transaction？有 AXI ID 后为什么还需要 Loopback？

## 回答

### 核心结论

> **这里的 Property 是接口在集成时声明的“能力开关/能力契约”，不是某一笔 transaction 的属性值。Property 决定接口必须具备哪些信号或识别哪些既有 opcode；具体 transaction 是否使用该能力，再由运行时信号或请求编码表达。**

## 1. Property、Signal 与 Transaction Encoding 的层次

以 Atomic 为例：

```text
Atomic_Transactions = TRUE
          ↓
接口声明支持 Atomic 能力
          ↓
接口必须具备/支持 AWATOP
          ↓
某一笔 transaction 的 AWATOP
编码具体 atomic operation
```

三者可这样区分：

| 层次 | 回答的问题 | 典型例子 |
| --- | --- | --- |
| Interface Property | 这个接口能否支持某类能力？ | `Atomic_Transactions = TRUE` |
| RTL Signal Set | 为该能力需要实现哪些 pin/sideband？ | `AWATOP`、`xPOISON`、`AxLOOP` |
| Per-transaction Encoding | 当前这一笔是否使用该能力、使用哪种语义？ | 某个 `AWATOP` 或 `AxSNOOP` 编码 |

Property 标注 `(No new signals)` 不表示“协议中无须表达”，而通常表示它复用已有的 `AxSNOOP`、request opcode 等编码，不需要新增一组 RTL pin。

## 2. 九类 Property 总览

| Property | 主要解决的问题 | 直观记忆 |
| --- | --- | --- |
| DeAllocation Transactions | 无用 cache line 继续占用容量 | 读完后顺便释放 cache line |
| Atomic Transactions | 多个 requester 竞争读—改—写 | 把“怎么改”送到数据所在处原子执行 |
| Loopback Signals | response 返回后恢复内部 transaction context | 给 transaction 挂一个原样返回的行李牌 |
| Poison | 已损坏数据在系统中的显式传播 | 数据继续走，但贴着“不可信”标签 |
| Unique ID Support | 下游判断是否存在同 ID 的在途 transaction | 告诉下游当前没有同 ID 兄弟 |
| Read Data Chunking | 某个慢数据块阻塞整笔 read 返回 | 哪个 128-bit chunk 准备好就先返回 |
| CMO On Read | 从 read request channel 发起 cache maintenance | Read channel 也能承载 CMO |
| Persist CMO | 数据必须到达掉电后仍受保证的位置 | 把脏数据推到持久化点 |
| Wakeup Signals | 下游可能处于低功耗状态 | transaction 到来前先通知下游唤醒 |

## 3. DeAllocation Transactions

普通 `ReadOnce` 关注取得最新数据；Deallocating Read 还表达：Device 读完后不希望相关 cache line 继续占据 cache capacity。

```text
Device Read + deallocation semantics
               ↓
       Coherent system
          ├─ 返回数据
          └─ 按相应语义处理 cached copies
```

Training 问答中涉及两个直观例子：

- `ReadOnceCleanInvalid (ROCI)`：读出数据，并以保留正确数据为前提清理、无效化相关 cache line。
- `ReadOnceMakeInvalid (ROMI)`：读出当前数据，并表达更强的 invalidation/deallocation 语义。

二者对 Dirty copy 的精确要求不能只靠名称推断，正式知识应以 ACE 规范的 request/response 与 data-source 规则为准。

它属于 cache resource/coherency 优化，不是地址 translation 功能。之所以可能写着 `No new signals`，是因为事务种类可以由既有请求 opcode/snoop encoding 表达。

## 4. Atomic Transactions

普通 `memory[x] += 1` 若拆成 Read 和 Write，两个 requester 可能同时读到旧值并覆盖彼此结果。Atomic transaction 将操作类型与操作数一起发送到下游可执行原子操作的位置：

```text
Atomic ADD 1
     ↓
读取旧值 → 运算 → 写入新值
     └──── 整体具备原子语义 ────┘
```

`AWATOP` 用于编码 Atomic Store、Load、Swap、Compare 等类别及 ADD、CLR、EOR、SET、MIN/MAX 等具体操作。

在 TBU 路径中，应这样理解：

```text
Device: IOVA + AWATOP
          ↓
TBU: 地址翻译并保持 atomic semantics
          ↓
System: PA + AWATOP
          ↓
具备支持能力的下游执行原子操作
```

TBU 通常不是最终完成运算的 Atomic ALU；它的关键责任是不能因地址翻译破坏 transaction 的原子语义。

## 5. Loopback Signals：AXI ID 不是唯一流水号

### 5.1 AXI ID 的核心用途

同一个 Master 可以同时发出多笔相同 ID 的 outstanding transaction：

```text
Read A → ARID = 3
Read B → ARID = 3
Read C → ARID = 3
```

AXI ID 更接近 ordering stream/tag，而不是全局或逐笔唯一流水号。相同 ID 的 transaction 必须遵守协议规定的 ordering relationship，但“同 ID”不表示“同一笔 transaction”。

Master 可能重复使用 ID，因为：

- 简单 DMA 只实现少量甚至单一 ID；
- Master 本来就希望一组 transaction 保持顺序；
- ID 数量与 Master 内部 outstanding table 的 entry 并非一一对应。

### 5.2 没有 Loopback 也能正确工作

若 Entry 12、27、41 的请求都使用 `ARID=3`，Master 可以维护：

```text
ID 3 tracking FIFO: 12 → 27 → 41
```

每收到一个 `RID=3` response，就按顺序取出相应 entry。因此 Loopback 不是 AXI 正确运行的必要条件。

### 5.3 Loopback 的真正用途

Loopback 允许 source 附加一段下游不解释、只需原样返回的信息：

```text
Request:  ARID=3, ARLOOP=27
Response: RID =3, RLOOP =27
```

Master 可以直接执行：

```text
RLOOP=27 → Outstanding_Table[27]
```

从而避免用 AXI ID 对内部状态做复杂的查找。Loopback value 可以用作 table index，但协议并不要求它必须唯一，这只是典型实现方式。

所以两者职责是：

| 字段 | 主要语义 | 谁需要理解 |
| --- | --- | --- |
| AXI ID | Ordering 与协议级 response association | AXI system/interconnect |
| LOOP | Source 自定义的 context/tracking tag | 中间组件只负责带回，source 解释 |

> **AXI ID 像“排序编号”，Loopback 更像 Master 自己挂的“行李牌”。**

## 6. Poison

当 ECC 等机制发现不可纠正错误时，系统有时仍需继续传递 data payload，同时用 `xPOISON` 表明数据已经损坏：

```text
Memory/Cache detects uncorrectable corruption
                  ↓
        Data + POISON=1
                  ↓
Interconnect → Consumer → containment/reporting
```

Poison 不是纠错机制，而是 corruption propagation mechanism：它不表示“数据已修复”，而表示“数据不可信，后续必须采取错误处理”。

这使错误状态能随数据到达真正的 consumer 或 fault-containment 单元，在 FuSa 系统中尤其重要。

## 7. Unique ID Support

`AxIDUNQ/xIDUNQ` 表达 unique-in-flight 信息。它不是说某个 ID 在整个 SoC 永久或全局唯一，而是告诉接收方：在相应协议范围和当前在途集合中，不存在需要考虑的同 ID transaction。

```text
ARID    = 7
ARIDUNQ = 1
        ↓
下游无需搜索、比较其他同 ID transaction
```

下游可以据此简化 ordering tracker、CAM/scoreboard lookup，降低 area、power 或 latency。

它与 Loopback 的方向不同：

- **IDUNQ** 帮助 downstream 判断同 ID ordering dependency。
- **LOOP** 帮助 requester 在 response 返回时找回自己的内部 context。

## 8. Read Data Chunking

普通 read transaction 的数据通常按规定顺序返回；如果前面的部分很慢，后面已经 ready 的数据也可能等待。

Read Data Chunking 允许在协议规则内，以 128-bit granule 标识并返回同一笔 read 的不同 chunk：

```text
逻辑顺序：Chunk0 Chunk1 Chunk2 Chunk3
准备顺序：Chunk2 Chunk3 Chunk0 Chunk1
```

相关信号的直观作用是：

| 信号 | 作用 |
| --- | --- |
| `ARCHUNKEN` | requester 表示该 read 允许 chunking |
| `RCHUNKV` | 当前 chunk metadata 有效 |
| `RCHUNKNUM` | 标识返回的是 transaction 中哪个 chunk |
| `RCHUNKSTRB` | 标识宽 `RDATA` beat 中哪些 128-bit chunk 有效 |

它优化的是**同一笔 read transaction 内部**的数据返回，不要与不同 transaction 之间的乱序混为一谈。

## 9. CMO On Read 与 Persist CMO

### CMO On Read

CMO（Cache Maintenance Operation）管理 cache line 的 clean、invalidate 等状态。`CMO_On_Read` 表示 ACE5-Lite interface 可以利用 Read request channel 承载相应 CMO。

`No new signals` 表示 CMO 类型复用已有 request opcode/snoop encoding，而不是不需要协议语义。

### Persist CMO

普通 Clean 关注把 Dirty data 推向一致性系统要求的位置；Persist CMO 进一步要求数据到达系统定义的持久化保证点，例如 Point of Persistence（PoP）或更深的 Point of Deep Persistence（PoDP）。

```text
Dirty cache line
       ↓ Clean/propagate
PoP / PoDP
       ↓
满足系统声明的 power-loss persistence model
```

> **普通 Clean 关注最新数据在一致性层次中的可见性，Persist 关注指定掉电模型下数据是否仍被保存。**

具体平台是否存在 persistent memory，以及 PoP/PoDP 位于哪里，属于系统实现定义。

## 10. Wakeup Signals

下游 interconnect 或 TBU 相邻逻辑可能处于 clock-gated/low-power 状态。`AWAKEUP` 允许 transaction source 在已有或即将出现活动时提前通知下游：

```text
Source ── AWAKEUP ──> Downstream leaves low-power state
   └──── AR/AW transaction 随后到来 ────┘
```

它把 bus activity 与 power-management handshake 连接起来。Training slide 中还把 TBM interface activity 与 `QACTIVE` 联系起来：接口存在活动时，低功耗控制不能把相关逻辑当作可安全关断的 idle 单元。

`AWAKEUP` 是活动/唤醒提示，并不等价于完整的 power-state transition；实际 clock/power 控制仍由系统低功耗基础设施完成。

## 11. 从 MMU-720AE TBU 角度统一理解

这组 Properties 并不是说 TBU 自己等同于 Cache、Atomic Engine、Persistent Memory Controller 或 Power Controller。

TBU 位于 inline ACE5-Lite data path 中，它在翻译地址的同时，还必须按照接口配置正确支持、保持或处理原 transaction 的高级语义：

```text
Device-side transaction
  ├─ IOVA
  ├─ Atomic / CMO / Deallocation semantics
  ├─ Poison
  ├─ ID / IDUNQ / LOOP
  ├─ Read chunk information
  └─ Wakeup/activity
             ↓
            TBU
  ├─ Translation and permission checks
  └─ Preserve/handle supported protocol semantics
             ↓
System-side transaction
```

因此，TBU 不能只理解为“把 IOVA 文本替换成 PA”的模块；它必须安全、透明地嵌入复杂的 ACE5-Lite transaction data path。

## 12. 证据边界

- **Training slide 明确内容**：列出的 interface Properties、相关新增信号，以及部分 `No new signals` 标注。
- **ACE/AXI 架构背景**：Atomic、Poison、IDUNQ、Loopback、Chunking、CMO 和 Wakeup 的协议用途。
- **待产品核实**：每个 Property 在当前 TBU 配置中的取值、所有合法编码、时序约束、信号宽度、错误行为和性能收益。

## 一句话总结

> **ACE5-Lite interface Property 声明 TBU 端口具备哪些高级协议能力；运行时再由 AWATOP、POISON、IDUNQ、LOOP、CHUNK、SNOOP/CMO 和 WAKEUP 等信号或编码描述具体 transaction。TBU 的职责是在完成地址翻译的同时，不丢失或破坏这些原有的原子性、一致性、错误传播、跟踪、性能和低功耗语义。**

## 相关记录

- [ACE-Lite Cache Stash 与目标 Node ID / LP ID](47_ACE-Lite_Cache_Stash与目标ID.md)
- [DMA 机制、SoC 应用与 SMMU 隔离](45_DMA机制_SoC应用与SMMU隔离.md)
- [LPD-500 低功耗分发与 Sequencer 行为](46_LPD-500低功耗分发与Sequencer.md)
