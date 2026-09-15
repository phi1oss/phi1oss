# PTW、HTTU、DPT 与 QTW/DVM 的流量分工

## 问题

如何理解下面这段 TRM 描述？

> The PTW interface is used only for PTW traffic, including HTTU updates, and DPT traffic, and these are not performed on the QTW/DVM interface.

## 核心结论

**这句话描述的是带专用 PTW 接口配置中的流量分工：PTW 接口承载 Translation Table Walk、HTTU 页表更新和 DPT 访问；这些流量不再走 QTW/DVM 接口。**

## 1. PTW Traffic：Translation Table Walk 流量

`PTW = Page Table Walk`。当 TCU 的相关缓存未命中时，需要从内存读取 Stage 1 或 Stage 2 页表描述符：

```text
TBU Translation Miss
         ↓
        TCU
         ↓
Translation Table Walk
         ↓
   PTW Interface
         ↓
读取 L0/L1/L2/L3 Descriptor
```

这里的 PTW traffic 专指用于完成地址翻译的 page-table walk，并不代表所有类型的 table access。

## 2. HTTU Update：硬件更新页表描述符

`HTTU = Hardware Translation Table Update`。

TCU 在进行 page-table walk 时，可能需要更新 translation descriptor 中的状态，例如：

```text
Access Flag
Dirty-related State
```

处理过程可以简化为：

```text
TCU 读取 PTE
     ↓
发现需要更新 AF / Dirty State
     ↓
AtomicCompare
     ↓
通过 PTW Interface 原子更新 PTE
```

HTTU update 与 page-table descriptor 紧密相关，因此也被归入 PTW 流量。

使用原子更新的原因是 CPU、Hypervisor 或 OS 可能同时修改同一个页表项。`AtomicCompare` 可以避免 TCU 使用旧值覆盖其他 agent 刚刚写入的新值。

## 3. DPT Traffic：Device Permission Table 访问

这里的 `DPT` 是：

```text
Device Permission Table
```

**DPT 不是 Dirty Page Tracking。**

DPT 用于描述设备是否有权访问相应的物理 granule。在支持 RME Device Assignment 的系统中，TCU 需要读取 DPT entry，并执行相关的设备访问权限检查；必要时还可能涉及相关状态更新。

这类 DPT memory traffic 也通过专用 PTW interface。

可以把 GPT 与 DPT 的关注点粗略区分为：

```text
GPT
→ 目标 Physical Granule 属于哪个 PAS？

DPT
→ 这个 Device 是否有权访问该 Physical Granule？
```

## 4. “Not Performed on QTW/DVM” 表示明确分流

在带专用 PTW interface 的配置中，流量被划分为两条路径：

```text
                         TCU
                          │
            ┌─────────────┴─────────────┐
            │                           │
       PTW Interface              QTW/DVM Interface
            │                           │
  Translation Table Walk          Queue Access
  HTTU Atomic Update              Configuration Table Walk
  DPT Traffic                     GPT Walk
                                  DVM Message
```

因此：

```text
PTW / HTTU / DPT
        ↓
固定走 PTW Interface
        ↓
不再走 QTW/DVM Interface
```

这里的 “not performed on the QTW/DVM interface” 不是说 QTW/DVM 完全不再访问内存，而是说上述三类流量已被分配给专用 PTW 通路。

## 5. 分流的意义：增加带宽和并行性

如果所有 TCU 内存流量都共用 QTW/DVM：

```text
Page-table Walk
Queue Access
Configuration Walk
GPT Walk
DVM Traffic
       ↓
共用 QTW/DVM Interface
```

高频 page-table walk 可能与 Queue、Configuration、GPT 和 DVM traffic 相互争用。

增加专用 PTW interface 后：

```text
地址翻译关键路径
PTW + HTTU + DPT
        ↓
专用 PTW Interface

其他管理和控制访问
Queue + Configuration + GPT + DVM
        ↓
QTW/DVM Interface
```

两条路径可以并行工作，从而：

- 减少接口争用
- 增加 table-walk bandwidth
- 降低 translation latency
- 提高 TCU 整体吞吐量

## TCUx1 与 TCUx2 的区别

```text
TCUx1
    没有独立 PTW Interface
    PTW、HTTU 等相关流量由 QTW 承载

TCUx2
    具有独立 PTW Interface
    PTW + HTTU + DPT 固定走 PTW
    不再走 QTW/DVM
```

因此，这段原文描述的“PTW traffic 不走 QTW/DVM”应结合带专用 PTW interface 的配置理解。

## 对比总结

| 流量 | 核心作用 | 接口路径（专用 PTW 配置） |
|---|---|---|
| PTW | 读取 Stage 1/Stage 2 Translation Table Descriptor | PTW Interface |
| HTTU | 原子更新 Access Flag、Dirty-related State 等页表状态 | PTW Interface |
| DPT | 读取 Device Permission Table，检查设备访问权限 | PTW Interface |
| Queue Access | Command/Event/PRI Queue 访问 | QTW/DVM Interface |
| Configuration Walk | 读取 STE/CD 等 translation configuration | QTW/DVM Interface |
| GPT Walk | 查询 physical granule 的 PAS 归属 | QTW/DVM Interface |
| DVM | Translation invalidate 和 synchronization 通信 | QTW/DVM Interface |

## 一句话总结

> **TCUx2 把地址翻译关键路径单独分流：Page-table walk、HTTU 原子更新和 DPT 访问全部走专用 PTW 接口，QTW/DVM 则继续负责 Queue、Configuration/GPT 访问及 DVM 通信。**
