# QTW/DVM 支持的六种 Transaction Type

## 问题

结合 coherency 和 atomic operation 的含义，如何提炼 MMU-720AE QTW/DVM 接口支持的六种 transaction type？

## 核心结论

QTW/DVM 接口支持的六种 transaction type 可以分成三组：

```text
Non-shareable Memory Access
    ReadNoSnoop
    WriteNoSnoop

Shareable / Coherent Memory Access
    ReadOnce
    WriteUnique

特殊控制 / 原子操作
    DVM Complete
    AtomicCompare
```

> **NoSnoop 表示不参与一致性；ReadOnce/WriteUnique 表示参与一致性；DVM Complete 表示 translation coherency 的同步完成；AtomicCompare 用于避免并发更新冲突。**

## 六种 Transaction Type

| Transaction Type | 核心含义 | 适用场景与重点 |
|---|---|---|
| **ReadNoSnoop** | 非一致性读 | 从 Non-shareable memory 读取，不 snoop 其他 cache |
| **WriteNoSnoop** | 非一致性写 | 向 Non-shareable memory 写入，不要求维护其他 cache 副本的一致性 |
| **ReadOnce** | 一致性读，但不长期缓存 | 从 Shareable memory 读取系统最新数据，必要时参与 snoop，但 TCU 不把它当作普通 cache line 长期保留 |
| **WriteUnique** | 一致性写 | 向 Shareable memory 写入，由 coherent interconnect 保证其他可能存在的旧 cache 副本不会与新值冲突 |
| **DVM Complete** | DVM 同步完成响应 | 不是普通数据访问；用来回应 DVM Sync，表示之前相关的 DVM TLBI/invalidate 已完成 |
| **AtomicCompare** | 原子 Compare-and-Swap | 原子执行“读→比较→条件写”；只有当前 memory value 仍等于预期旧值时才更新，避免 CPU/TCU 并发修改造成 lost update；在 MMU-720AE 中主要用于 HTTU |

## 1. ReadNoSnoop 与 WriteNoSnoop

这两种 transaction 用于 **Non-shareable memory**：

```text
ReadNoSnoop
    ↓
普通非一致性读取
不查询其他 Manager 的 Cache

WriteNoSnoop
    ↓
普通非一致性写入
不维护其他 Cache 副本的一致性
```

它们适用于不需要进入 coherent domain 的 Queue、Table 或 GPT 等 memory access。

需要注意：`NoSnoop` 描述的是这次访问不参与 cache coherency，并不是指访问对象一定属于某一种固定的数据结构。

## 2. ReadOnce 与 WriteUnique

这两种 transaction 用于 **Shareable/Coherent memory**。

### ReadOnce

```text
TCU 发起 ReadOnce
        ↓
Coherent Interconnect
        ↓
必要时查询其他 Cache
        ↓
TCU 获得系统中的最新数据
```

TCU 需要读取 coherent domain 中的最新值，但不会像 CPU Cache 那样长期持有一份普通 coherent cache line。

### WriteUnique

```text
TCU 发起 WriteUnique
        ↓
Coherent Interconnect
        ↓
处理可能冲突的旧 Cached Copy
        ↓
使新值对 Coherent Observers 正确可见
```

`WriteUnique` 不表示 TCU 自己逐个修改其他 Cache，而是由 coherent interconnect 保证系统中不会继续保留与新值冲突的旧副本。

因此：

```text
ReadOnce
→ 一致性地获得最新数据

WriteUnique
→ 一致性地写入新数据
```

## 3. DVM Complete

`DVM Complete` 不是普通 memory read/write，而是 translation coherency 流程中的 completion message。

```text
CPU / Initiator
      │ DVM TLB Invalidate
      ▼
Coherent Interconnect
      │
      ▼
     TCU
      │ 清除相关 Translation State
      ▼

CPU / Initiator
      │ DVM Sync
      ▼
     TCU
      │ 等待前面的 DVM 操作完成
      ▼
DVM Complete
```

它表达的是：

> **之前相关的 DVM TLBI/invalidate 已经处理完成，可以结束本次同步。**

## 4. AtomicCompare

`AtomicCompare` 是原子的 Compare-and-Swap（CAS）：

```text
old = Memory[address]

if old == compare:
    Memory[address] = swap

return old
```

读取、比较和条件写入由系统保证为一个不可被竞争访问破坏的原子过程。

在 MMU-720AE 中，它主要用于 HTTU 更新 translation-table descriptor，例如 Access Flag 或 dirty-related state：

```text
CPU --------┐
            ├── 可能同时修改同一个 Page Table Entry
TCU --------┘
```

如果 TCU 使用普通 read-modify-write，可能覆盖 CPU 同时写入的新值；使用 AtomicCompare 后，只有 descriptor 仍等于 TCU 预期的旧值时才会更新，否则不写并重新处理，从而避免 lost update。

## TCUx1 与 TCUx2 的差异

MMU-720AE 对 HTTU atomic traffic 的路径有一项实现差异：

```text
TCUx1
HTTU AtomicCompare → QTW Interface

TCUx2
HTTU Atomic Traffic → PTW Interface
```

因此，QTW/DVM 列表中的 `AtomicCompare` 只适用于 **TCUx1**；TCUx2 的相关 HTTU atomic traffic 改走 PTW。

## 对比记忆

```text
                         QTW/DVM
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
   Non-shareable       Shareable /       特殊操作
   Memory Access       Coherent Access        │
          │                 │                 │
  ReadNoSnoop          ReadOnce         DVM Complete
  WriteNoSnoop         WriteUnique      AtomicCompare
```

## 一句话总结

> **ReadNoSnoop/WriteNoSnoop 用于非共享内存访问，ReadOnce/WriteUnique 用于需要 coherent semantics 的共享内存访问，DVM Complete 用于确认 translation invalidation 同步完成，AtomicCompare 则用于 HTTU 对页表描述符进行无竞争的原子更新。**
