# PCIe Root Complex、Root Port 与 Bifurcation

记录日期：2026-09-04  
记录端：手机 / GitHub；本地整理于 2026-09-06  
来源：[Root Complex 原记录](https://github.com/phi1oss/phi1oss/blob/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93/00_Inbox/2026-09-04_1533_PCIe_Root_Complex%E7%B3%BB%E7%BB%9F%E5%AE%9A%E4%BD%8D%E4%B8%8ESMMU%E5%85%B3%E7%B3%BB.md)、[Bifurcation 原记录](https://github.com/phi1oss/phi1oss/blob/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93/00_Inbox/2026-09-04_1551_PCIe_bifurcation%E6%A6%82%E5%BF%B5.md)、[Root Port 原记录](https://github.com/phi1oss/phi1oss/blob/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93/00_Inbox/2026-09-04_1552_PCIe_Root_Port%E6%A6%82%E5%BF%B5.md)  
状态：Inbox；Requester ID→StreamID 映射、bifurcation 后的端口数量及 TBU/LTI 拓扑需以具体 SoC 集成为准

## 问题

PCIe Root Complex 和 Root Port 分别是什么？PCIe bifurcation 改变了什么？它们与 DMA、SMMU、ATS 和 PRI 有什么关系？

## 回答

### 核心结论

> **Root Complex 是 PCIe 主机侧的整体根节点，Root Port 是它向下连接 PCIe hierarchy 的一个具体端口；bifurcation 把一组宽链路拆成多条独立的窄链路，因此系统会面对更多独立端口和 transaction sources，SMMU 的身份映射与接入拓扑也必须相应匹配。**

### 1. Root Complex 与 Root Port

| 概念 | 系统定位 | 主要职责 |
| --- | --- | --- |
| Root Complex（RC） | PCIe hierarchy 的 Host/Root 侧整体功能 | 连接 CPU/Memory/NoC 与 PCIe；发起配置访问；承接 PCIe transaction、interrupt 和 requester identity |
| Root Port（RP） | Root Complex 向下的一条具体 PCIe 端口 | 面向一条 link、switch hierarchy 或 Endpoint 收发 TLP，并把 transaction 接入 SoC |

可以用一个类比记忆：**Root Complex 像整个总部，Root Port 是其中一个对外出口。**

### 2. 枚举与身份识别

系统软件通过 Root Complex 提供的配置访问机制枚举 PCIe hierarchy，读取或配置：

- Vendor ID、Device ID、Class Code。
- BAR 和地址窗口。
- MSI/MSI-X。
- ATS、PRI、PASID 等 capability。

Root Complex 提供事务和访问机制，软件负责枚举策略与资源分配。

PCIe transaction 的 Requester ID（通常与 BDF 身份相关）进入 SoC 后，需要由平台集成逻辑形成或映射为 SMMU 使用的 StreamID：

```text
PCIe Requester ID / BDF
          ↓  平台定义的映射
       StreamID
          ↓
       SMMU STE
```

因此不能笼统地说“BDF 永远直接等于 SID”。具体编码和映射位置取决于 Root Complex、互连和 SMMU 的集成。

### 3. Root Complex、DMA 与 SMMU 的分工

PCIe Device 可以包含自己的 DMA engine。一次典型访问可概括为：

```text
PCIe Device DMA
      ↓ TLP
Root Port / Root Complex
      ↓ SoC transaction + requester identity
SMMU
      ↓ translation + permission check
Memory / target
```

- **Device DMA** 决定搬什么数据。
- **Root Complex** 负责把 PCIe transaction 接入 SoC，并保留/转换必要身份与属性。
- **SMMU** 决定该 requester 使用哪个 translation context，以及允许访问哪里。

一句话说，Root Complex 管“怎样进入 SoC”，SMMU 管“进入后能访问哪里”。

### 4. ATS 与 PRI 中的 Root Complex

ATS translation request 的概念路径是：

```text
Device → Root Port / Root Complex → SMMU/TCU
Device ← Root Port / Root Complex ← Translation result
```

在 MMU-720AE 的相关集成中，Root Port 可通过 DTI-ATS 使用 TCU 的 translation service。Root Complex 是协议和路径桥梁，translation 的配置、page table walk、permission check 与结果生成仍由 SMMU 侧负责。

PRI page request 也会经过 Root Complex 到达 SMMU/软件处理路径；Root Complex 本身不是缺页处理器。

MSI/MSI-X 则通常表现为 PCIe memory write，经 Root Complex 进入 SoC 的中断目标。Root Complex 负责传递，不等同于 GIC。

### 5. Bifurcation 为什么会影响 SMMU 集成

PCIe bifurcation 是把一组宽 lane 拆成多组独立窄 link，例如：

```text
x16 → x8 + x8
x16 → x4 + x4 + x4 + x4
```

它不只是“每条链路带宽变小”，还可能让控制器对外呈现多个独立 Root Port/link 实例。每个端口下的 Device 都会产生独立 transaction 和 requester identity。

Training 中的集成启示是：

- **Inline/ACE-Lite TBU 路径**：数据流经过 TBU；多个独立端口可能要求多条数据通路或相应汇聚结构。
- **LTI/lookaside 类路径**：可以集中处理多个端口发来的 translation request，而实际数据走独立的 NoC 数据路径。

这说明 bifurcation 会影响端口数量、SID 映射、translation request 汇聚方式和数据路径规划。精确拓扑必须以 PCIe controller 配置、SoC integration diagram 和 MMU-720AE 实例连接为准。

## 一句话总结

> **Root Complex 是 PCIe 主机侧整体，Root Port 是一个具体出口；bifurcation 把宽链路拆成多个独立出口，因而 SMMU 必须正确接入每个端口并区分其 requester identity。**

## 相关记录

- [TBU、TCU、DTI 与 DTI-ATS](01_TBU_TCU_DTI与DTI-ATS概述.md)
- [PCIe ATS 与 ATC 工作流程](35_PCIe_ATS与ATC工作流程.md)
- [PRI 缺页恢复与无 PRI 时的退化路径](37_PRI缺页恢复与无PRI退化路径.md)
- [SMMU 与 I/O Device 的三种集成方式](39_SMMU三种设备集成方式.md)
