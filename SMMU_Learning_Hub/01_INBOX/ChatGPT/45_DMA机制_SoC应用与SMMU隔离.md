# DMA 机制、SoC 应用与 SMMU 隔离

记录日期：2026-09-04  
记录端：手机 / GitHub；本地整理于 2026-09-06  
来源：[GitHub 手机端原记录](https://github.com/phi1oss/phi1oss/blob/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93/00_Inbox/2026-09-04_1534_DMA%E6%9C%BA%E5%88%B6%E4%B8%8ESoC%E5%85%B8%E5%9E%8B%E5%BA%94%E7%94%A8.md)  
状态：Inbox；coherency、cache maintenance、descriptor translation 与 fault handling 需结合具体 DMA IP、互连和 SMMU 集成核实

## 问题

DMA 的本质工作机制是什么？它在 SoC 中有哪些典型应用？为什么 DMA requester 还需要 SMMU？

## 回答

### 核心结论

> **DMA 负责在总线上主动搬运数据，使 CPU 不必逐字节或逐拍参与；SMMU 不负责搬运数据，而是为 DMA 地址做翻译、权限检查和隔离。**

### 1. DMA 到底省掉了什么

没有 DMA 时，CPU 往往需要反复执行 load/store，把数据从外设寄存器或 FIFO 搬到内存。使用 DMA 时，CPU 通常只负责：

1. 配置源地址、目标地址、长度、方向、burst 和 channel。
2. 启动 DMA。
3. 在完成或出错后处理中断/状态。

真正的数据传输由 DMA controller 或 Device 内部 DMA engine 作为 bus master/requester 发起。因此“无需 CPU 参与”更准确地说是：**CPU 不再参与每个数据 beat，但仍负责配置、同步和异常处理。**

### 2. 常见传输方向与配套机制

DMA 常见于：

- Peripheral → Memory，例如 UART RX、Camera capture、Ethernet RX。
- Memory → Peripheral，例如 UART TX、Audio playback、Display output。
- Memory → Memory，例如大块拷贝或数据重排。

常见配套机制包括：

- **FIFO**：吸收外设速率和总线 burst 之间的差异。
- **Peripheral handshake**：只在外设 ready/valid 条件满足时取数或送数，避免 overflow、underflow 或重复消费。
- **Scatter-Gather**：DMA 读取 descriptor chain，在不连续 buffer 之间持续传输。
- **Ping-pong/Circular buffer**：适合持续音频、视频等流式数据。

### 3. DMA 不是某一个固定 IP

“DMA”描述的是一种主动读写内存的能力，可能由不同主体实现：

- SoC 中的通用 DMA controller。
- Camera、Audio、Ethernet 等专用外设内部 DMA engine。
- PCIe Endpoint（例如 NVMe 或 NIC）内部 DMA engine。

所以 PCIe Device 做 DMA，并不表示它一定使用 SoC 内部的通用 DMA controller。

### 4. DMA 为什么需要 SMMU

DMA engine 是 requester，能够主动发起 memory access。如果缺少隔离，配置错误或失控的 Device 可能访问任意物理内存。

```text
CPU VA       → CPU MMU → PA
Device IOVA  → SMMU    → PA
```

SMMU 为 DMA 请求提供：

- IOVA/VA 到 PA 的地址翻译。
- Read/Write permission check。
- 不同 Device、Process、VM 之间的地址空间隔离。
- Fault detection 与软件可见的错误报告。

SVA 场景中，Device 可以使用进程虚拟地址，但仍需 PASID/SSID、CD、页表共享和生命周期管理；这不改变“SMMU 负责翻译与隔离”的基本分工。

### 5. Camera DMA 的完整例子

```text
软件分配 frame buffer
        ↓
建立 SMMU IOVA → PA mapping
        ↓
把 IOVA、长度、stride 配置给 Camera DMA
        ↓
Camera DMA 作为 requester 发起 write
        ↓
SMMU translation + permission check
        ↓
frame 写入 physical memory
        ↓
DMA completion interrupt
```

如果映射不存在或权限不允许，SMMU 报告 fault；DMA 是否停止、重试、丢弃后续数据或进入错误状态，则由 SMMU 配置与 DMA IP 的错误处理机制共同决定。

### 6. RTL/SoC 集成时应检查什么

- requester 使用 PA、IOVA 还是 Process VA。
- SID/SSID 如何产生，访问是否经过 TBU/SMMU。
- burst、outstanding、FIFO depth 和 backpressure 是否匹配。
- descriptor fetch 与 payload access 是否使用相同 translation context。
- DMA buffer 是否 coherent；若不是，软件何时 clean/invalidate cache。
- fault 后 DMA 如何停机、排空、恢复或通知软件。

## 一句话总结

> **DMA 决定“搬什么、怎么搬”，SMMU 决定“这些地址翻译到哪里、是否允许搬”；两者分别解决性能和隔离问题。**

## 相关记录

- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [PCIe ATS 与 ATC 工作流程](35_PCIe_ATS与ATC工作流程.md)
- [SVA：共享虚拟地址与进程—设备绑定](38_SVA共享虚拟地址与进程设备绑定.md)
- [SMMU 与 I/O Device 的三种集成方式](39_SMMU三种设备集成方式.md)
