# DGW 与 AGW：两类 GPC 路径及 GPT Cache

## 问题

如何理解下面这段 TRM 描述？为什么 TCU 中需要两套 GPT Cache？

> Two GPT caches exist in the TCU for the following:
>
> - Client-originated AXI Granule Protection Check (GPC) checks, DTI GPC Wrapper (DGW).
> - SMMU-originated GPC checks, AXI GPC Wrapper (AGW).

## 核心结论

**两套 GPT Cache 的区别不在于缓存的 GPT 格式不同，而在于它们分别服务两类来源完全不同的物理访问：DGW 负责 client-originated access，AGW 负责 SMMU 自己产生的 access。**

```text
外部设备 / Client 发起的访问
              ↓
             DGW
              ↓
         GPT Cache #1

SMMU / TCU 自己发起的访问
              ↓
             AGW
              ↓
         GPT Cache #2
```

## 1. DGW：检查 Client-originated access

这里的 client 是使用 SMMU translation 服务的外部 requester 或 device，例如：

```text
PCIe Device
GPU
DMA Engine
其他 I/O Requester
```

设备发起 DMA 后，SMMU 首先完成地址翻译：

```text
Device
   │ IOVA
   ▼
TBU / SMMU
   │ Translation
   ▼
Physical Address
```

在支持 RME/GPC 的系统中，得到 PA 后还不能立即认为访问合法，还需要检查：

```text
这个 PA 所在的 granule 属于哪个 PAS？
当前访问是否有权进入这个 PAS？
```

这就是 **client-originated GPC**。MMU-720AE 中，这条检查路径由：

```text
DGW = DTI GPC Wrapper
```

负责，并使用自己的一套 GPT Cache。

因此，DGW 回答的核心问题是：

> **“外部设备翻译得到这个 PA 后，是否有权访问它所属的 PAS？”**

## 2. AGW：检查 SMMU-originated access

TCU 为了完成 SMMU 自身的工作，也会主动访问内存，例如：

```text
TCU
 ├─ 读取 Stream Table / STE
 ├─ 读取 Context Descriptor
 ├─ 读取 Page Table
 ├─ 读取 GPT
 ├─ 读取 Command Queue
 ├─ 写入 Event Queue
 ├─ 写入 PRI Queue
 └─ 写入 MSI
```

这些 transaction 不是某个设备 DMA 原封不动地通过 SMMU，而是：

> **SMMU 为完成内部工作而主动生成的 memory transaction。**

因此，它们属于 **SMMU-originated access**。

这些访问同样不能绕开物理安全检查，所以由：

```text
AGW = AXI GPC Wrapper
```

执行 GPC，并使用另一套 GPT Cache。

因此，AGW 回答的核心问题是：

> **“SMMU 自己发出的 AXI 内存访问，是否有权访问目标 PA 所属的 PAS？”**

## 3. 为什么两类访问都必须执行 GPC？

假设某段物理内存属于 Realm PAS：

```text
PA  = 0x8000_0000 ～ 0x8FFF_FFFF
PAS = Realm
```

如果系统只检查：

```text
Device → Realm Memory
```

却不检查：

```text
TCU → Realm Memory
```

那么 SMMU 自己产生的 page-table walk、configuration-table walk 或 queue access，就可能成为绕过物理安全边界的路径。

因此，安全规则必须覆盖所有最终的物理访问：

```text
无论是谁产生 Physical Access
              ↓
只要目标是受 GPT 管理的 physical granule
              ↓
就必须执行相应的 GPC
```

这就是 MMU-720AE 同时支持 client-originated GPC 和 SMMU-originated GPC 的原因。

## 4. Wrapper 和 GPT Cache 分别做什么？

`Wrapper` 可以理解成包围在某条访问路径周围的一层 GPC 检查逻辑。DGW 和 AGW 都不是 GPT 本身，其基本处理过程是：

```text
收到 Physical Access
          ↓
查询对应的 GPT Cache
          ↓ miss
必要时执行 GPT Walk
          ↓
获得目标 granule 的 PAS 信息
          ↓
执行 Granule Protection Check
          ↓
允许访问 / 拒绝访问
```

两套 GPT Cache 分别为两条 GPC datapath 缓存 GPT Walk 结果，减少重复的 GPT 表访问：

```text
DGW
 └─ 缓存 client-originated GPC 所需的 GPT 信息

AGW
 └─ 缓存 SMMU-originated GPC 所需的 GPT 信息
```

## 对比总结

| 对比项 | DGW | AGW |
|---|---|---|
| 全称 | DTI GPC Wrapper | AXI GPC Wrapper |
| 访问来源 | 外部 client/device | SMMU/TCU 自身 |
| 典型访问 | 设备地址翻译后的物理访问检查 | 查表、访问队列、MSI 等内部内存访问 |
| 检查目标 | Client-originated GPC | SMMU-originated GPC |
| GPT Cache | 使用 client 路径的 GPT Cache | 使用 SMMU 路径的 GPT Cache |
| 共同点 | 查询 GPT/PAS 信息并完成 GPC | 查询 GPT/PAS 信息并完成 GPC |

## 一句话总结

> **DGW 管“设备翻译后要访问哪里的 GPC”，AGW 管“SMMU 自己为了查表、访问队列等产生的内存访问的 GPC”；两者都查询 GPT，只是访问来源不同。**
