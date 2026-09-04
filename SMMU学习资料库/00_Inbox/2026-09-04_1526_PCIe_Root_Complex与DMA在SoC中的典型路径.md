---
received_at: 2026-09-04 15:26 +08:00
source: SMMU聊天框
status: inbox
topic: PCIe Root Complex / DMA / SMMU
source_material: 2026-09-04 上午对话
---

# PCIe Root Complex 与 DMA 在 SoC 中的典型路径

> 本条按手机端 Inbox 方式收录。已与当前 `00_Inbox` 中已有的 `PRI Queue`、`SVA`、`Stream Table` 相关文件去重：这些主题虽有交叉，但本条聚焦 **PCIe Root Complex 的系统定位、DMA 的基本机制，以及 DMA/PCIe/SMMU 的典型数据路径**，不重复展开 PRI/SVA 的完整流程。

## 原问题 1：PCIe Root Complex 是什么？

### 核心结论

**PCIe Root Complex（RC）是 PCIe hierarchy 的根节点，负责连接 CPU/Memory System 与 PCIe Endpoint。它既是 PCIe 设备进入 SoC 的入口，也是配置访问、TLP 转换、Requester 身份管理以及 ATS/PRI/MSI 等机制的重要桥接点。**

可以先记成：

```text
CPU / Memory System
        │
        ▼
  PCIe Root Complex
        │
     PCIe Link
        │
        ▼
   PCIe Endpoint
```

典型 Endpoint 包括：

- NVMe SSD
- NIC
- GPU / Accelerator
- 其他 PCIe 外设

### Root Complex 解决的第一个问题：协议与层级桥接

SoC 内部常见的是 AXI/ACE/CHI 等互联协议，而 PCIe 链路上传输的是 PCIe TLP。

因此 Root Complex 需要在两边之间完成 transaction bridge：

```text
CPU / SoC side
AXI / CHI transaction
        │
        ▼
Root Complex
        │
        ▼
PCIe Memory / Config TLP
        │
        ▼
PCIe Endpoint
```

反方向：

```text
PCIe Endpoint
   │ Memory Read/Write TLP
   ▼
Root Complex
   │ SoC-side transaction
   ▼
SMMU / Interconnect / Memory
```

### Root Complex 与 Root Port 的区别

```text
Root Complex
├─ Root Port 0 → Endpoint
├─ Root Port 1 → Endpoint
└─ Root Port 2 → PCIe Switch
```

- **Root Complex**：整个 PCIe Host/Root 功能块。
- **Root Port**：Root Complex 向下连接某条 PCIe Link 的端口。

一句话：

> **Root Complex 是“整套根节点”，Root Port 是“其中的一条出口”。**

### Root Complex 与 PCIe 枚举

系统启动后，Host Software 通过 Root Complex 发起 Configuration Read/Write，读取 Endpoint 的：

```text
Vendor ID
Device ID
Class Code
BAR
MSI / MSI-X capability
ATS capability
PRI capability
PASID capability
...
```

然后软件完成：

```text
Bus / Device / Function 编号
BAR 地址分配
Memory Window 配置
Interrupt 配置
ATS / PRI / PASID 等 feature enable
```

所以：

> **Root Complex 是 PCIe 枚举的硬件入口，Firmware/OS 是枚举的控制者。**

### Requester ID 与 SMMU StreamID

PCIe Device 发起 transaction 时通常带有 Requester ID，常与 BDF（Bus:Device:Function）相关。

在 Arm SoC 中，Root Complex / IOMMU 集成逻辑通常需要把 PCIe requester identity 映射成 SMMU 可以识别的 StreamID：

```text
PCIe Requester ID / BDF
        ↓
Root Complex / SID mapping
        ↓
StreamID
        ↓
SMMU
        ↓
STE
        ↓
Translation Context
```

因此可以这样分工：

```text
Root Complex：
“这是哪个 PCIe Device 发来的请求？”

SMMU：
“这个 Device 的地址该怎么翻，它有没有权限访问？”
```

### Root Complex 与 ATS / PRI

ATS：

```text
PCIe Device
   │ ATS Translation Request
   ▼
Root Complex
   │ DTI-ATS / SMMU translation request
   ▼
SMMU / TCU
   │ Translation Result
   ▼
Root Complex
   │ ATS Completion
   ▼
PCIe Device
```

PRI：

```text
PCIe Device
   │ PRI Page Request
   ▼
Root Complex
   │
   ▼
SMMU
   │
   ▼
PRIQ → Software
```

所以 Root Complex 是：

> **PCIe ATS/PRI 协议与 Arm SMMU translation system 之间的重要桥接节点。**

### Root Complex 与 MSI / GIC

PCIe Device 发 MSI 本质上是一个 PCIe Memory Write：

```text
PCIe Device
   │ MSI Memory Write TLP
   ▼
Root Complex
   ▼
SoC Interconnect
   ▼
GIC ITS / MSI target
```

Root Complex 不是 GIC/ITS，但它是 PCIe MSI write 进入 Arm SoC 的入口。

### Root Complex 面试式总结

> **PCIe Root Complex 是 PCIe hierarchy 的根节点，连接处理器/内存系统与 PCIe Endpoint，负责 PCIe 枚举、配置访问、PCIe TLP 与 SoC 内部 transaction 的桥接，并管理 Requester identity。对于带 SMMU 的 Arm SoC，Root Complex 通常把 PCIe Requester ID 映射为 StreamID，并把 DMA、ATS、PRI、MSI 等请求导入相应的 SMMU、Memory System 或 GIC 路径。**

一句话记忆：

> **Root Complex 管“PCIe 设备怎么进入 SoC”；SMMU 管“这个设备进入之后能访问哪块内存”。**

---

## 原问题 2：DMA 是什么？在 SoC 中有哪些典型应用？

### 核心结论

**DMA（Direct Memory Access）允许外设或 DMA Engine 在不由 CPU 逐个数据 beat 搬运的情况下，直接在 Memory 与 Peripheral 之间或 Memory 与 Memory 之间传输数据。CPU 负责配置任务，DMA 负责真正搬数据。**

没有 DMA：

```text
Peripheral
   ↓ read
  CPU
   ↓ write
 Memory
```

有 DMA：

```text
Peripheral
    │
    ▼
DMA Controller
    │
    ▼
 Memory
```

### DMA Controller 本质上是 Bus Master / Requester

在 AXI SoC 中，DMA 通常能主动发起 AXI transaction，因此从系统视角，它和 CPU/GPU/PCIe 等一样属于 requester/master：

```text
CPU
DMA
GPU
PCIe
USB
Ethernet
   ↓
AXI / CHI Interconnect
   ↓
Memory / Peripheral
```

所以 DMA 并不只是一个“寄存器块”，它具备主动访问系统地址空间的能力。

### 三类典型 DMA 传输

#### 1. Peripheral → Memory

例如 UART RX：

```text
UART FIFO
   ↓
DMA
   ↓
DDR Buffer
```

#### 2. Memory → Peripheral

例如 Audio Playback：

```text
DDR PCM Buffer
      ↓
     DMA
      ↓
I2S / Audio Controller
```

#### 3. Memory → Memory

```text
DDR Region A
    ↓
   DMA
    ↓
DDR Region B
```

典型用途包括 memcpy、图像 buffer copy、数据重排等。

### 一次 DMA 传输怎么启动

CPU 通常配置：

```text
Source Address
Destination Address
Transfer Length
Direction
Burst Size
Channel
Interrupt / Completion mode
```

然后：

```text
CPU
 ↓ 写 DMA 寄存器 / Descriptor
DMA Enable
 ↓
DMA 自动发起 bus transaction
 ↓
完成后 Interrupt CPU
```

因此：

> **CPU 负责“安排搬运”，DMA 负责“真正搬运”。**

### DMA 为什么常带 FIFO

外设与 DDR 的速率经常不同，例如：

```text
DDR 很快
UART 很慢
```

DMA 通过内部 FIFO 解耦：

```text
DDR
 ↓ burst read
DMA FIFO
 ↓ 按外设速率输出
UART
```

反向接收同理：

```text
UART
 ↓
DMA FIFO
 ↓ burst write
DDR
```

### SoC 中的典型 DMA 场景

#### UART

```text
UART RX FIFO
   ↓
DMA
   ↓
DDR Buffer
```

减少 CPU 逐字节中断和搬运。

#### Camera / Sensor

```text
Camera Sensor
   ↓
MIPI CSI / Camera Controller
   ↓
DMA
   ↓
DDR Frame Buffer
```

适合高带宽连续数据流。

#### Audio

```text
DDR Audio Buffer
   ↓
DMA
   ↓
I2S / Codec
```

常结合 Ping-Pong Buffer 保证连续播放/录音。

#### Ethernet

```text
Network
   ↓
Ethernet MAC
   ↓
DMA Engine
   ↓
DDR Packet Buffer
```

高吞吐网卡几乎不可能由 CPU 逐 byte 搬运。

#### PCIe / NVMe / NIC

很多高性能设备内部本身就带 DMA engine：

```text
PCIe Endpoint
   │ DMA Read / Write
   ▼
Root Complex
   ▼
SMMU
   ▼
DDR
```

此时并不需要额外经过 SoC 中的通用 DMA Controller。

因此广义 DMA 更应该理解为：

> **一种 Device/Engine 主动访问 Memory 并完成数据搬运的机制。**

### DMA 与 SMMU 的关系

DMA requester 可以主动访问系统内存，因此必须有隔离机制。

如果没有 SMMU：

```text
DMA Device
   ↓
Physical Address
   ↓
DDR
```

设备理论上可能触碰很大的物理地址范围。

带 SMMU 后：

```text
DMA Device
   │ IOVA
   ▼
SMMU
   │ Translation + Permission Check
   ▼
PA
   ▼
DDR
```

于是软件可以只给 Device 映射有限的 buffer：

```text
IOVA 0x1000
   ↓
SMMU
   ↓
PA 0x8000_1000
```

而未映射地址会产生 translation fault。

### 为什么 Device 地址常叫 IOVA

```text
CPU：
VA → CPU MMU → PA

Device：
IOVA → SMMU → PA
```

所以 IOVA 可以理解成 Device 侧的虚拟地址。

传统 DMA：

```text
CPU VA
 ↓ Driver / DMA mapping
Device IOVA
 ↓ SMMU
PA
```

而 SVA 则进一步让：

```text
CPU Process VA
≈
Device 使用的 VA
```

这部分已在现有 SVA Inbox 条目中单独收录，因此本条不重复展开。

### Scatter-Gather / Descriptor DMA

真实系统中 buffer 不一定物理连续，DMA 常通过 descriptor chain 描述多个片段：

```text
Descriptor 0
src=A, dst=B, len=4KB

Descriptor 1
src=C, dst=D, len=8KB

Descriptor 2
...
```

DMA 自动：

```text
读 Descriptor
 ↓
执行 Transfer
 ↓
读下一个 Descriptor
```

这就是 Scatter-Gather DMA。

### Peripheral Handshake

外设 DMA 通常还需要握手：

```text
Peripheral
   │ DMA Request
   ▼
DMA
   │ transfer
   ▼
Peripheral
   │ DMA Ack / request deassert
```

这样 DMA 不会无节制地读写速度较慢的外设 FIFO。

### DMA 面试式总结

> **DMA 是一种允许外设或 DMA Controller 在不由 CPU 逐数据搬运的情况下直接访问系统内存的机制。DMA Controller 通常作为 AXI Master，通过源地址、目的地址、长度、burst、descriptor 等配置完成 Peripheral↔Memory 或 Memory↔Memory 传输。典型应用包括 UART、SPI、Audio、Camera、Ethernet 和 PCIe。对于带 SMMU 的 SoC，DMA requester 通常使用 IOVA 发起访问，由 SMMU 完成 IOVA→PA translation 和 permission check，从而实现 DMA isolation。**

一句话记忆：

> **CPU 决定“搬什么、搬到哪”，DMA 负责“真正搬”；SMMU 决定“这个 DMA 到底能不能访问那块内存”。**

---

## Root Complex、DMA、SMMU 三者如何串起来

可以用一条 PCIe NIC/NVMe DMA 路径理解：

```text
PCIe Endpoint
   │
   │ DMA Memory Read/Write TLP
   ▼
Root Complex
   │
   │ Requester ID → StreamID
   │ PCIe TLP → SoC transaction
   ▼
SMMU
   │
   │ IOVA → PA
   │ Permission Check
   ▼
Memory System / DDR
```

三者分工：

```text
DMA / PCIe Device
→ 发起数据搬运

Root Complex
→ 让 PCIe transaction 进入 SoC，并提供/转换 requester identity

SMMU
→ 决定这个 requester 的地址如何翻译、允许访问哪些内存
```

---

## RTL / 集成视角关注点

1. PCIe Requester ID 到 SMMU StreamID 的映射关系。
2. Root Complex 发出的 SoC-side transaction 是否携带正确的 SID / security / translation attributes。
3. DMA 是否作为独立 AXI Master，是否需要通过 SMMU/TBU。
4. DMA descriptor / data buffer 的地址是 PA、IOVA 还是 Process VA，需要和 software driver 模型一致。
5. Cache coherency：DMA buffer 是否 coherent，是否需要 cache maintenance。
6. DMA burst、FIFO、outstanding transaction、AXI ID 等性能参数。
7. Peripheral DMA request/ack handshake 是否会发生丢请求、重复请求或 backpressure 问题。
8. PCIe ATS/SVA 场景下不要把 pre-translated traffic 与普通 untranslated DMA 混淆。

---

## 验证与调试关注点

```text
Case 1: PCIe Device DMA + valid SMMU mapping
→ Root Complex 正确映射 Requester ID/SID
→ SMMU translation success
→ DDR 数据正确

Case 2: DMA 访问未映射 IOVA
→ SMMU translation fault
→ Device 不得越权访问目标 PA

Case 3: 两个 PCIe Device 使用不同 SID
→ 各自只能访问分配给自己的 IOVA/PA 范围

Case 4: Scatter-Gather DMA
→ descriptor 链顺序、长度和地址正确

Case 5: Peripheral DMA handshake
→ FIFO 满/空、backpressure 下无数据丢失

Case 6: DMA completion interrupt
→ 完成后中断和状态寄存器一致
```

---

## 一句话总览

> **DMA 是“谁来搬数据”，Root Complex 是“PCIe 设备怎样进入 SoC”，SMMU 是“设备进入 SoC 后能访问哪里”。**
