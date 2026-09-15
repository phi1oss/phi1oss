# AMBA Atomic Transaction 与 Exclusive Access

记录日期：2026-09-07  
记录端：手机 / GitHub；本地整理于 2026-09-07  
来源：[GitHub 手机端原记录](https://github.com/phi1oss/phi1oss/blob/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93/00_Inbox/AMBA_Atomic_Transaction__2026-09-07.md)  
状态：Inbox；`AWATOP` 精确编码、不同 Atomic 类别的返回通道，以及与 ACE coherency/cache-line ownership 的交互待结合 AXI5/ACE5 规范校验

## 问题

什么是 AMBA/AXI5 Atomic Transaction？它为什么能够避免多个 Master 修改同一共享地址时的竞争？它与 Exclusive Access 有何区别，经过 MMU-720AE TBU 时又如何处理？

## 回答

### 核心结论

> **Atomic Transaction 让 Master 直接把 ADD、SWAP、COMPARE 等操作发送给下游，由具备支持能力的组件不可分割地完成 Read-Modify-Write；Exclusive Access 则仍由 Master 自己完成 Read 和 Write，并依靠系统检测两者之间是否发生冲突。**

## 1. 为什么需要 Atomic Transaction

假设两个 Master 同时对共享计数器加一。若使用彼此分离的普通 Read 和 Write：

```text
Master A: Read 10
Master B: Read 10
Master A: Write 11
Master B: Write 11

最终值 = 11    // 丢失了一次更新，正确结果应为 12
```

问题在于 Read→Modify→Write 不是一个不可分割的整体，其他 requester 可以在中间观察或修改同一位置。

使用 Atomic ADD 后：

```text
Master A: Atomic ADD 1
          ↓
Target: old=10 → new=11 → write 11

Master B: Atomic ADD 1
          ↓
Target: old=11 → new=12 → write 12
```

下游把每次操作作为原子单元执行，避免两个 Master 都基于同一个旧值更新。

## 2. 核心信号与语义

- `AWATOP`：编码 Atomic operation 的类型和操作语义。
- `WDATA`：通常携带 operand、比较值或交换值等操作数据。
- 下游具备 Atomic 支持的组件：不可分割地执行读取旧值、运算和写入新值。
- Read response：某些 Atomic 类别还会把修改前的旧值返回给 Master。

常见类别可先建立为：

```text
AtomicStore
AtomicLoad
AtomicSwap
AtomicCompare
```

其中 AtomicLoad/Store 还可以表达 ADD、CLR、EOR、SET、signed/unsigned MIN/MAX 等操作。具体 `AWATOP` 编码、operand 格式和 response 规则必须查对应 AXI5 版本。

## 3. Atomic 与 Exclusive Access 的区别

### Exclusive Access

```text
Master: Exclusive Read
          ↓
Master 本地计算新值
          ↓
Master: Exclusive Write
          ↓
System 检查期间是否被其他访问破坏了 exclusive 状态
          ↓
成功，或失败后由 Master 重试
```

它属于 **requester-side read/modify/write + conflict detection/retry model**。

### Atomic Transaction

```text
Master: Atomic ADD/SWAP/COMPARE + operand
          ↓
下游在数据所在位置一次完成原子操作
```

它属于 **subordinate/downstream-side atomic operation model**。

| 对比项 | Exclusive Access | Atomic Transaction |
| --- | --- | --- |
| Read/Modify/Write 由谁组织 | Master | 支持 Atomic 的下游组件 |
| 总线上的主要形式 | Exclusive Read + Exclusive Write | 一笔带 `AWATOP` 的 Atomic request，并按类别返回 response |
| 冲突后的典型处理 | Exclusive Write 失败，Master 重试 | 下游保证一次 Atomic operation 的不可分割性 |
| 数据是否往返 Master 再计算 | 通常是 | 操作被送到数据所在位置执行 |

一句话记忆：

> **Exclusive 是“我自己读改写，你帮我检查有没有冲突”；Atomic 是“我告诉你怎么改，你一次性替我完成”。**

## 4. 在 MMU-720AE / TBU 中如何理解

```text
Device
  │ Atomic request + IOVA + AWATOP/operand
  ▼
TBU
  ├─ 根据 translation context 完成 IOVA → PA
  ├─ 执行相应 permission/fault processing
  └─ 保持 Atomic transaction 的协议语义
  ▼
System
  │ PA + Atomic operation
  ▼
支持 Atomic 的下游组件执行操作
```

TBU 通常不是最终执行 Atomic 运算的 ALU。它位于 inline data path 中，核心责任是：

1. 正确翻译 Atomic request 的地址。
2. 不因翻译、拆分或转发破坏 Atomic semantic。
3. 按产品和协议规则处理 translation fault、response 与相关 ordering。

因此 `Atomic_Transactions` Property 表示 TBU interface 支持此类 transaction 通过，而不是说 TBU 自身承担所有 Atomic 运算。

## 5. 尚需核实的边界

- AXI5 `AWATOP` 的完整编码分类。
- AtomicLoad、AtomicStore、AtomicSwap、AtomicCompare 是否及如何返回旧值。
- operand 宽度、burst/size/alignment 等合法条件。
- Atomic transaction 与 ACE coherency、cache line ownership 和 snoop 的具体交互。
- translation fault 时是否产生任何可见 memory side effect，以及 TBU 的 response 行为。

## 一句话总结

> **AMBA Atomic Transaction 用 `AWATOP` 把 Read-Modify-Write 操作送到下游原子执行；Exclusive Access 则由 Master 自己读改写并在失败时重试。TBU 负责地址翻译和保持 Atomic 语义，不等于执行运算的 Atomic ALU。**

## 相关记录

- [ACE5-Lite 接口 Properties：能力声明与事务语义](48_ACE5-Lite接口Properties与事务语义.md)
- [DMA 机制、SoC 应用与 SMMU 隔离](45_DMA机制_SoC应用与SMMU隔离.md)
