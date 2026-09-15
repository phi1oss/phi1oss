# DTI 协议分层、消息类型与完整 Translation 流程

记录日期：2026-09-07  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，面向初学者的 DTI 协议概念解释  
状态：Inbox；DTI message 的精确名称、字段、方向、token/connection 管理与各通道支持范围待结合 DTI 规范和 MMU-720AE TRM 校验

## 问题

作为没有接触过 DTI 的初学者，应该如何理解 DTI 协议？它为什么存在、传递什么信息，与 DMA 数据路径有什么区别？DTI-TBU 和 DTI-ATS 又分别服务于什么场景？

## 回答

### 核心结论

> **DTI（Distributed Translation Interface）是分布式 SMMU 组件之间传递 translation 与 control message 的协议：TBU 本地无法完成翻译时通过 DTI 请求 TCU 服务，TCU 也通过 DTI 向 TBU 分发失效和同步控制；真正的 Device/DMA data 不走 DTI。**

## 1. 为什么分布式 SMMU 需要 DTI

MMU-720AE 不是把所有 translation logic 都集中在 Device 数据路径上的一个单体模块，而是把职责分给靠近 requester 的 TBU 与中央 TCU：

```text
Device
  ↓
TBU                         // 前端：inline data path、本地 cache
  │
  │ DTI
  ▼
TCU                         // 后端：集中 translation/control
  ↓
Stream Table / CD Table / Translation Tables
```

- **TBU**：接收 Device transaction，查本地 translation cache/TLB；命中时快速形成输出访问。
- **TCU**：处理复杂 configuration lookup、translation walk、backend caching 和全局维护控制。
- **DTI**：连接两者，使分布式前端能够使用中央 translation service。

如果没有 DTI，TBU miss 后就无法把 `SID + SSID + Address + attributes` 等请求上下文交给 TCU，也无法收到 translation result。

## 2. 最典型的 DTI-TBU Translation 流程

```text
Device transaction
      ↓
     TBU
      ↓
Local TLB/cache lookup
   ┌───────┴────────┐
   │                │
  Hit              Miss
   │                │
本地完成      DTI Translation Request
                    ↓
                   TCU
                    ↓
       configuration/translation lookup
                    ↓
        DTI Translation Response
                    ↓
                   TBU
                    ↓
          fill cache + continue access
```

因此：

> **DTI-TBU 是 TBU 与 TCU 之间的 translation/control 通道；TBU 是前台代理，TCU 是后台翻译中心。**

“TBU miss”并不必然等价于每次都完整访问内存页表：TCU 可能先命中自己的 configuration、translation 或 walk cache；只有相应后端 lookup 也 miss 时，才需要访问内存中的表结构。

## 3. DTI 与 DMA 数据路径必须分开

真正的 memory data path 是：

```text
Device
  ↓ Address + WDATA/RDATA
TBU
  ↓ translated transaction
System Interconnect
  ↓
Memory / Target
```

DTI control path 是：

```text
TBU / PCIe Root Port
  ↓ translation/control messages
DTI Interconnect
  ↓
TCU
```

DTI 传递的是类似以下语义：

- “请翻译这个地址”；
- “这是 translation result/fault”；
- “使某些 cached translation 失效”；
- “确认之前的维护已经完成”；
- “建立或解除 translation service connection”。

它不承载 Device 的普通 `WDATA/RDATA` payload。

> **数据走 AXI/ACE datapath，translation/control message 走 DTI。**

## 4. DTI 消息可以先分成哪些类别

### 4.1 Translation Request / Response

```text
TBU → TCU：请求某个 address/context 的 translation
TCU → TBU：返回成功结果、attributes 或 fault indication
```

这是 DTI-TBU 最核心的服务。

### 4.2 Invalidation

软件通过 CMDQ 发出 TLBI/CFGI 等维护命令后，TCU 必须让相关 TBU 的缓存状态失效：

```text
Software → CMDQ → TCU
                    ↓
             DTI invalidation
                    ↓
                  TBU(s)
```

这样可以避免 TBU 继续使用已经过期的 translation/configuration。

### 4.3 Synchronization / Completion

在 TLBI/CFGI 后执行 `CMD_SYNC` 时，TCU 需要确认相关分布式动作已经完成：

```text
TCU → TBU(s)：同步/完成边界相关消息
TBU(s) → TCU：完成响应
TCU → Software：CMD_SYNC completion
```

其意义是建立维护操作的全局完成点，而不仅是“命令已从 CMDQ 取走”。

### 4.4 Connection Management

TBU 启动、复位或重新加入系统时，需要与 TCU 建立可用的 DTI service relationship，可能涉及连接状态和 translation resource/token 管理：

```text
TBU connect
    ↓
TCU accepts/denies and assigns usable resources
    ↓
translation service becomes available
```

具体 connect/disconnect message、token 数量与失败响应属于 DTI/MMU-720AE 实现细节。

### 4.5 ATS / Page Request 相关消息

这一类必须按接口类型区分。PCIe ATS/PRI 场景通常通过 DTI-ATS 把 Root Port 的 translation/page-request control traffic 送到 TCU；不能笼统地把所有 Page Request 都画成普通 TBU 数据路径上的消息。

```text
PCIe Device
  ↓ ATS Translation Request / PRI Page Request
Root Port
  ↓
DTI-ATS
  ↓
TCU / SMMU software-visible queues
```

精确的 message support 和响应路径需分别查 DTI-TBU、DTI-ATS 规范与产品配置。

## 5. DTI-TBU 与 DTI-ATS 的区别

| 对比项 | DTI-TBU | DTI-ATS |
| --- | --- | --- |
| 典型 requester | TBU | PCIe Root Port/ATS bridge |
| 主要目的 | TBU miss translation、cache maintenance、同步和连接管理 | 为 PCIe ATS/PRI 提供 translation/page-request control service |
| 服务方 | TCU | TCU |
| 是否承载 DMA payload | 否 | 否 |

两者都使用 TCU 的 translation/control 能力，但发起方和 message set 不完全相同。

## 6. DTI 与 DTI Interconnect 的关系

DTI 定义 message 的功能语义；DTI interconnect 负责在多个 requester 和 TCU 端口之间搬运这些 message。

```text
TBU0 ─┐
TBU1 ─┼─> DTI switch/sizer/register slice ─> TCU
ATS  ─┘
```

Switch、sizer 和 register slice 可以完成路由、宽度适配和时序切分，但本身不执行 page-table walk，也不决定 translation result。

## 7. DTI 为什么使用 AXI5-Stream

MMU-720AE 中可以用协议分层理解 DTI：

```text
SMMU translation/control semantics
                ↓
             DTI message
                ↓
        AXI5-Stream transport
                ↓
             RTL signals
```

- **DTI** 回答：“这一包 message 是 translation request、response、invalidate 还是 sync？”
- **AXI5-Stream** 回答：“message 如何进行 ready/valid 传输、分拍和流控？”

DTI 传输的是 message，而不是对某个 memory-mapped address 执行普通 read/write，因此使用 stream transport 比把每类控制消息伪装成 AXI memory transaction 更自然。

## 8. 一次 TLB Miss 的完整例子

Device 发出：

```text
IOVA = 0x1000
SID  = 20
```

TBU 本地 lookup：

```text
IOVA + SID
   ↓
MicroTLB miss
   ↓
Main TLB miss
```

TBU 发出 DTI request：

```text
TBU
  │ DTI Translation Request
  │ SID=20, Address=0x1000, context/attributes...
  ▼
TCU
```

TCU 处理：

```text
SID → STE
       ↓ Stage 1 enabled 时
SSID → CD
       ↓
Translation lookup/walk
       ↓
PA + permissions + output attributes
```

结果通过 DTI 返回 TBU：

```text
TCU → DTI Translation Response → TBU
                                  ↓
                          fill translation cache
                                  ↓
                       IOVA 0x1000 → PA 0x80001000
```

下一笔匹配相同 translation context 和地址范围的 transaction 若本地命中，就不再发送新的 DTI Translation Request。

## 9. 初学者最容易混淆的边界

1. **DTI 不是页表**：它只是传递 translation/control message。
2. **DTI 不是 DMA data path**：它不搬运普通 Device payload。
3. **DTI interconnect 不做 translation**：真正决定结果的是 TBU/TCU 及其 table/cache logic。
4. **DTI request 不等于必然 page-table walk**：TCU backend cache 可能命中。
5. **DTI-TBU 不等于 DTI-ATS**：发起方和支持的 message 类型不同。
6. **AXI5-Stream 不等于 DTI 语义**：前者是 transport，后者定义 SMMU message。

## 10. 证据边界

- **当前问答和既有 Training/TRM 主线**：DTI 连接 TBU/ATS requester 与 TCU，承载 translation/control，不承载 DMA payload。
- **架构性解释**：本地 TLB miss、backend lookup/walk、invalidation 与 `CMD_SYNC` completion 的因果关系。
- **待规范核实**：所有 message 的正式名称、字段、方向、顺序规则、token/connection 语义以及 DTI-TBU/DTI-ATS 各自支持的精确集合。

## 一句话总结

> **DTI 是分布式 SMMU 的 translation/control 消息协议：TBU miss 时用 DTI-TBU 请求 TCU，TCU 用它下发失效并收集同步完成；PCIe Root Port 则通过 DTI-ATS 使用 TCU 的 ATS/PRI 服务。DTI message 由 AXI5-Stream 承载，但真正的 DMA data 走独立的 AXI/ACE 数据路径。**

## 相关记录

- [TBU、TCU、DTI 与 DTI-ATS](01_TBU_TCU_DTI与DTI-ATS概述.md)
- [PCIe ATS 与 ATC 工作流程](35_PCIe_ATS与ATC工作流程.md)
- [PRI 缺页恢复与无 PRI 时的退化路径](37_PRI缺页恢复与无PRI退化路径.md)
- [CMD_SYNC 的本质与完成边界](33_CMD_SYNC的本质与完成边界.md)
