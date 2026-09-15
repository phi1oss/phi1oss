# Command Queue、Event Queue 与 PRI Queue

## 问题

MMU-720AE 中的 Command Queue、Event Queue 和 PRI Queue 分别有什么作用？三者的数据方向和协作关系是什么？

## 核心结论

**三类 Queue 都是位于 memory 中的环形队列，但数据流方向和用途不同：**

```text
Command Queue：Software → SMMU
Event Queue：  SMMU → Software
PRI Queue：    PCIe Device → SMMU → Software
```

> **Command Queue 是软件给 SMMU 下任务，Event Queue 是 SMMU 给软件报告事件，PRI Queue 是 PCIe 设备请求软件准备某个当前不可用的 page。**

## 1. Command Queue：软件给 SMMU 下管理命令 

Command Queue 是 **CPU/SMMU Driver 写、TCU 读** 的队列：

```text
CPU / SMMU Driver
        │
        │ 写入 Command
        ▼
┌─────────────────────┐
│    Command Queue    │  ← 位于 Memory
└─────────────────────┘
        │
        │ TCU Queue Manager 读取
        ▼
       SMMU
```

它传输的不是普通 DMA transaction，而是发给 SMMU 的管理命令，例如：

```text
TLB Invalidate
Configuration Cache Invalidate
Synchronization
PRI Response
……
```

例如，软件修改页表后，需要清除旧的 translation：

```text
Software 修改 Page Table
           ↓
旧 Translation 可能仍在 TLB 中
           ↓
Software 向 Command Queue 写入 TLBI Command
           ↓
TCU 读取并执行 Command
           ↓
旧 TLB Entry 被 Invalidate
```

Command Queue 使用典型的 producer/consumer 机制：

```text
Software                 TCU
Producer                Consumer
   │                        │
   ▼                        ▼
CMDQ_PROD              CMDQ_CONS

        Command Queue
┌────┬────┬────┬────┬────┐
│CMD0│CMD1│CMD2│    │    │
└────┴────┴────┴────┴────┘
```

`SMMU_CMDQ_PROD` 和 `SMMU_CMDQ_CONS` 分别记录 producer 和 consumer 的位置。

> **Command Queue = Software → SMMU 的异步任务队列。**

## 2. Event Queue：SMMU 向软件报告事件

Event Queue 的方向与 Command Queue 相反：

```text
       SMMU / TCU
           │
           │ 生成 Event Record
           ▼
┌─────────────────────┐
│     Event Queue     │  ← 位于 Memory
└─────────────────────┘
           │
           │ Software 读取
           ▼
     SMMU Driver / OS
```

它主要用于记录 **SMMU 架构和地址翻译层面的 event/fault**。

例如：

```text
Device
  │ IOVA = 0x1234_5000
  ▼
SMMU Translation
  ▼
Page Table Walk
  │
  └── Descriptor Invalid
          ↓
    Translation Fault
          ↓
      Event Record
          ↓
      Event Queue
          ↓
       Software
```

Event Record 可以让软件知道：

```text
哪个 Stream 出现问题
哪个地址发生 Fault
发生了什么类型的 Translation/Event
```

需要把它与 RAS 硬件错误区分开：

```text
Event Queue
    ↓
SMMU 功能和地址翻译层面的事件
例如 Translation Fault

TCU_ERRSTATUS / FMU
    ↓
硬件可靠性错误
例如 ECC、Parity、RAM Corruption
```

> **Event Queue = SMMU → Software 的事件和故障报告队列。**

## 3. PRI Queue：设备请求软件准备 Page

`PRI = Page Request Interface`，主要服务于支持 PCIe ATS/PRI 的设备。

它解决的问题是：

> **PCIe Device 想访问一个合法的虚拟地址，但对应 page 当前还没有可用的 physical backing 或有效 translation。设备通过 PRI 请求软件把这个 page 准备出来。**

例如：

```text
PCIe Device 需要访问
IOVA = 0x1234_5000
          ↓
当前 Page Table 中对应项
Invalid / Not Present
```

### 3.1 发现 Page 当前不可用

设备首先请求 translation：

```text
PCIe Endpoint
      │ ATS Translation Request
      ▼
PCIe Root Port
      │ DTI-ATS
      ▼
     TCU
      │ Translation Walk
      ▼
  Page Table
      │
  Not Present
```

### 3.2 设备发出 PRI Page Request

```text
PCIe Endpoint
      │ PRI Page Request
      ▼
PCIe Root Port
      ▼
     SMMU
      │ 生成 Page Request Record
      ▼
┌─────────────────┐
│    PRI Queue    │  ← 位于 Memory
└─────────────────┘
      │
更新 PRIQ_PROD
      │
产生 PRIQ Interrupt
      ▼
OS / IOMMU Driver
```

PRI Queue 中的 Page Request Record 可以让软件识别：

```text
哪个 Device / Stream
哪个 Process / Substream
哪个 Page Address
请求什么类型的访问
```

### 3.3 软件准备 Page

```text
OS / IOMMU Driver
        │
        ▼
检查访问是否合法
        │
   ┌────┴────┐
   │         │
 合法      不合法
   │         │
   ▼         ▼
分配或调入   不建立 Mapping
Physical Page
   │
   ▼
更新 Page Table
   │
   ▼
执行必要的同步操作
```

### 3.4 软件通过 Command Queue 返回 PRI Response

Page 准备完成后，软件需要把处理结果返回给设备。此时会再次使用 Command Queue：

```text
OS / IOMMU Driver
        │ PRI Response Command
        ▼
┌─────────────────┐
│  Command Queue  │
└─────────────────┘
        │
        ▼
       TCU
        │
        ▼
PCIe Root Port
        │ PRI Response
        ▼
PCIe Endpoint
```

### 3.5 设备重新请求 Translation 并执行 DMA

```text
PCIe Endpoint
      │ ATS Translation Request
      ▼
     SMMU
      │ Page Table 已经有效
      ▼
IOVA → PA Translation 成功
      │
      ▼
Device ATC 缓存 Translation
      │
      ▼
执行真正的 DMA Read / Write
```

### 完整 PRI 流程

```text
PCIe Device
     │ Page 暂时不可用
     ▼
PRI Page Request
     ▼
   SMMU
     ▼
┌─────────────┐
│  PRI Queue  │
└─────────────┘
     ▼
OS / IOMMU Driver
     │
     ├── 分配或调入 Physical Page
     └── 更新 Page Table
     ▼
┌──────────────────┐
│  Command Queue   │
│  PRI Response    │
└──────────────────┘
     ▼
   SMMU
     ▼
PCIe Device
     │ Retry Translation
     ▼
Translation Success
     ▼
执行真正的 DMA
```

> **PRI Queue 是保存 PCIe Page Request 的 memory-resident queue。设备请求软件准备当前不可用的 page；软件根据 PRIQ 中的信息准备物理页并更新页表，再通过 Command Queue 返回 PRI Response，最后设备重新进行 translation 并继续 DMA。**

## 三类 Queue 的整体关系

```text
                    CPU / OS / SMMU Driver
                      │              ▲
                      │ Command      │ Event
                      ▼              │
             ┌─────────────┐  ┌─────────────┐
             │ Command Q   │  │  Event Q    │
             └─────────────┘  └─────────────┘
                      │              ▲
                      ▼              │
                   ┌────────────────────┐
                   │     SMMU / TCU     │
                   └────────────────────┘
                            ▲
                            │ Page Request
                     ┌─────────────┐
                     │    PRI Q    │
                     └─────────────┘
                            ▲
                            │
                      PCIe Device
```

## 对比总结

| Queue | Producer | Consumer | 核心用途 |
|---|---|---|---|
| Command Queue | Software | SMMU/TCU | 软件向 SMMU 下达管理命令 |
| Event Queue | SMMU/TCU | Software | SMMU 向软件报告事件或 translation fault |
| PRI Queue | SMMU 代表 PCIe Device 写入 | Software | 向软件传递设备的 Page Request |

## 一句话总结

> **Command Queue：软件给 SMMU 下任务；Event Queue：SMMU 给软件报事件；PRI Queue：PCIe 设备通过 SMMU 请求软件准备缺失的 page。**
