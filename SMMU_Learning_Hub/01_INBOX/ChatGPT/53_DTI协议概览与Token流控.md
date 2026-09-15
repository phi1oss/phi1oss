# DTI 协议概览与 Token 流控

记录日期：2026-09-08
记录端：本地

## 来源与整理范围

来源：[SMMU 对话](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)，2026-09-08 的相关问答。

本篇为 Inbox 来源记录（Capture），不是已查证晋升的正式知识。问题按原意凝练；回答保留原文，仅整理标题层级、排版及移除不可解析的引用标记。原回答中的“Slide 明确写”“TRM 明确说明”等为原答表述，本次未重新打开规范逐条复核。截图图像未随本篇保存，可回原会话查看；截图若来自培训视频，则为“培训视频截图，时间点未知”，原会话未提供视频时间戳。

## DTI protocol：第 81 页解析

### 问题

MMU 培训文档第 81 页的 DTI protocol 如何理解？

### 回答

第 81 页这张 **DTI protocol** slide，核心其实是在回答三个问题：

> **DTI 是干什么的？谁和谁用？里面传什么？**

培训资料把 DTI 定义为一种 **message-based protocol**，用于 SMMU 内部组件以及外部 translation agent 与 TCU 之间通信。

##### 1. 为什么需要 DTI

MMU-720AE 是分布式结构：

```text
Device
  ↓
 TBU              TBU
  │                │
  └──── DTI ───────┤
                   ↓
                  TCU
```

TBU 靠近 Device，TCU 掌握中心的 translation/configuration 能力。

例如 TBU 本地 TLB miss：

```text
Device request
    ↓
TBU TLB miss
    ↓
DTI Translation Request
    ↓
TCU
    ↓
完成 translation
    ↓
DTI Translation Response
    ↓
TBU
```

所以可以先记：

> **DTI 是 TBU/ATS requester 和 TCU 之间的“翻译控制消息通道”，不是 DMA 数据通道。**

---

##### 2. `message-based protocol` 怎么理解

DTI 不是普通 AXI 那种：

```text
Read address X
Write address Y
```

而是发送有明确含义的消息，例如：

```text
TRANSLATION_REQ
TRANSLATION_RESP

INV_REQ
SYNC_REQ

PAGE_REQ

REG_READ
REG_WRITE
```

每个 message 都是在告诉另一端：

> “我要你执行一个 SMMU 管理/translation 操作。”

所以它更像：

```text
DTI message
= 消息类型 + 该消息需要的参数
```

---

##### 3. DTI 自己不规定物理线怎么传

Slide 说：

> DTI messages are carried over the physical BAS protocol in Arm SMMUs. 

也就是分层：

```text
SMMU功能
   ↓
DTI
“消息是什么意思”
   ↓
BAS / AXI-Stream
“消息怎么在硬件线上传”
```

所以：

> **DTI 是功能协议，BAS 是 transport。**

MMU-720AE TRM 也明确说明 DTI 可以承载在 AXI5-Stream 上，而且上下行各有一条 AXI5-Stream interface。

---

#### 4. 两种 DTI 子协议

##### DTI-TBU

```text
TBU ←──── DTI-TBU ────→ TCU
```

主要服务正常 SMMU translation：

```text
TBU miss
→ 请求TCU translation
```

也包括：

- invalidation
- synchronization
- TBU register access
- connection management

---

##### DTI-ATS

```text
PCIe Root Port
      ↕
    DTI-ATS
      ↕
     TCU
```

这是给 PCIe ATS 用的。

例如：

```text
PCIe Device
 ↓ ATS Translation Request
PCIe Root Port
 ↓ DTI-ATS
TCU
 ↓
translation result
```

所以：

> **DTI-TBU 的 requester 是 TBU；DTI-ATS 的 requester 是 PCIe Root Port/ATS manager。**

培训资料对此也是这样区分的。

---

#### 5. DTI 有哪些主要 message？

这页把它们分成五类。

```text
① Connection / Disconnection
   → 建立和关闭DTI连接

② Translation Request / Response
   → 请求地址翻译

③ Invalidation / Synchronization
   → TLB/config cache失效，并确认完成

④ Page Request
   → PRI/缺页恢复相关

⑤ Register Access
   → TCU通过DTI访问TBU寄存器
```

其中你现在最重要的是先掌握前三种。

---

##### 6. 为什么还需要 Connection？

DTI 不是 TBU 上电以后随便就可以无限发请求。

TBU首先：

```text
TBU
 ↓ REQ_CONNECT
TCU
 ↓
分配translation token等资源
 ↓
CONNECTED
```

之后才能正常发 translation request。

所以 connection 过程实际上还承担：

> **建立通信关系 + 协商/分配 flow-control resource。**

下一页讲的 `Translation Token` 就是沿着这里来的。

---

#### 7. MMU-720AE 用哪个版本？

培训资料里说明：

```text
MMU S3 r0 / MMU-720AE
→ DTI-TBU v3
```

而外部 ATS manager 可以支持：

```text
DTI-ATS v1
DTI-ATS v2
DTI-ATS v3
```

具体产品支持哪些版本要看实现。

---

##### 一句话记忆

> **DTI = SMMU 分布式组件之间的“控制消息协议”：TBU miss 找 TCU 翻地址、TCU 通知 TBU 做 TLBI、PCIe ATS 找 TCU 请求 translation，这些都通过 DTI message 完成；真正的 DMA 数据不走 DTI。**

下一页的 **DTI flow control / Token** 正好是在回答另一个自然问题：

> **“既然 TBU 可以不断给 TCU 发 Translation Request，怎么防止把 TCU 的处理能力撑爆？”**

答案就是 Translation Token。

---

## Token：容量控制与请求身份的区别

### 问题

DTI 如何用 Translation Token 和 Invalidation Token 控制在途请求？Token 与 TID 有什么区别？

### 回答

这页讲的是 **DTI 的 credit/token-based flow control**。你可以把 Token 直接理解成：

> **“我允许你同时欠我多少笔请求。”**

没有 Token，就不能继续发对应类型的 request。

###### 1. 为什么 DTI 需要 Token

假设 TBU 连续 TLB miss：

```text
TBU
 ↓ Translation Req #1
 ↓ Translation Req #2
 ↓ Translation Req #3
 ↓ ...
TCU
```

TCU 内部能同时处理的 translation 数量有限。

如果 TBU 无限发送：

```text
Request速度 > TCU处理能力
              ↓
          Buffer overflow
```

所以 TCU 先给 TBU 一定数量的 **Translation Token**。

例如：

```text
TCU给 TBU：4 tokens
```

那么最多允许：

```text
4笔 outstanding translation request
```

---

##### 2. Translation Token

Slide：

> Translation tokens – limit outstanding translation requests

例如初始：

```text
Translation Token = 3
```

TBU发送：

```text
TRANS_REQ #0
Token: 3 → 2

TRANS_REQ #1
Token: 2 → 1

TRANS_REQ #2
Token: 1 → 0
```

此时即使又发生 TLB miss：

```text
Token = 0
   ↓
不能再发 TRANS_REQ
```

等 TCU 返回：

```text
TRANS_RESP
```

这笔 request 对应的 token 被释放：

```text
Token: 0 → 1
```

TBU 才能继续发。

所以：

> **Request 消耗 Token，Response 把 Token 还回来。**

---

##### 3. Invalidation Token

方向正好反过来。

Translation request 通常是：

```text
TBU → TCU
```

所以 Translation Token 是：

> **TCU 给 TBU 的 credit。**

而 invalidation 通常：

```text
TCU → TBU
```

例如：

```text
CMD_TLBI
  ↓
TCU
  ↓ DTI invalidate
TBU
```

这时候需要保护的是 **TBU 能同时接收多少个 invalidation**。

因此：

> **Invalidation Token 由 TBU（或 PCIe Root Port）授予 TCU。**

比如：

```text
TBU给TCU：1个 invalidation token
```

那么：

```text
TCU
 ↓ INV_REQ
TBU

token → 0
```

直到：

```text
TBU → acknowledgement
```

token 才返回。

---

#### 4. 两种 Token 的方向一定要记清楚

```text
Translation Request

TBU  ─────────→ TCU
        ↑
Translation Token
由 TCU 提供
```

```text
Invalidation Request

TCU  ─────────→ TBU
        ↑
Invalidation Token
由 TBU 提供
```

一句话：

> **谁接收 Request，谁就提供 Token。**

这个规律最好记。

---

#### 5. Connection 阶段为什么要协商 Token

右边时序图最开始：

```text
DISCONNECTED

TBU / PCIe RP
     ↓
CONDIS_REQ(connect, ...)
     ↓
TCU

     ↑
CONDIS_ACK(connect, ...)
```

connection 不只是：

> “你好，我们连上了。”

还会建立后续 flow-control resource。

例如 TBU 会请求一定数量的 translation token。

Slide 特别提到：

```text
max_tok_trans
```

它控制：

> **TBU 在 connection request 中最多请求多少个 Translation Token。**

例如：

```text
max_tok_trans = 4
```

TBU希望：

```text
最多同时存在4笔 translation request
```

但有一个硬限制：

> **TBU 请求的 Translation Token 数不能超过 TCU 实际 translation slots 的能力。**

因为：

```text
TCU只有4个translation slots

却给TBU 16 tokens
        ↓
允许16 outstanding
        ↓
TCU根本接不住
```

Token 本质上就是对真实硬件资源的抽象。

---

#### 6. 右边时序图怎么读

连接成功后：

```text
TRANS_REQ(A0,TID0)
TRANS_REQ(A1,TID1)
TRANS_REQ(A2,TID2)
```

TBU 可以连续发送多笔 translation request，只要：

```text
还有 Translation Token
```

然后 TCU response：

```text
TRANS_RESP(...)
```

可以不是立刻一个 Request 对一个 Response 紧跟着返回。

这说明：

> **DTI 支持多个 outstanding translation requests。**

Token 就控制 outstanding 上限。

图中的 `TID0/TID1/TID2` 则用于区分这些在途请求。

---

#### 7. 为什么同时需要 Token 和 TID？

这两个东西不要混。

```text
Token
→ 控制“最多能同时发多少笔”
```

```text
TID
→ 区分“返回的是哪一笔”
```

例如：

```text
Token = 4
```

表示最多允许：

```text
4 outstanding
```

而：

```text
TID0
TID1
TID2
TID3
```

负责追踪这 4 笔请求。

类似我们前面讨论 AXI：

> **Token 管容量，TID 管身份。**

---

#### 8. Disconnect 时为什么要归还 Token

Slide：

> Tokens are returned when TBU or PCIe Root Port requests disconnection

因为 disconnect 后，这条 DTI connection 不再继续工作：

```text
CONNECTED
   ↓
CONDIS_REQ(disconnect)
   ↓
等待相关事务收尾
   ↓
CONDIS_ACK(disconnect)
   ↓
DISCONNECTED
```

之前为这条 connection 分配的 translation/invalidation resource 就应该释放。

因此：

> **Token 是 connection 生命周期内的资源，不是永久属于某个 TBU。**

---

##### 一句话总结

> **DTI Token 就是 credit-based flow control：Translation Token 由 TCU 提供，限制 TBU/PCIe RP 对 TCU 的 outstanding translation request；Invalidation Token 由 TBU/PCIe RP 提供，限制 TCU 对它们的 outstanding invalidation。Request 消耗 Token，Response/Ack 归还 Token。**

一句话记忆：

> **谁接收 Request，谁发 Token；Token 决定“你最多可以欠我几笔请求”。**

## 一句话总结

DTI 规定翻译控制消息的含义；Token 控制在途请求数量，TID 区分请求身份。

## 整理注与待核实边界

第一个问题明确指向 MMU.pdf 第 81 页；第二个问题为截图解析，未确认页码。版本支持、max_tok_trans 与实际 slots 的约束保留原回答口径，待对照对应版本资料核实。

## 关联记录

- [DTI协议分层_消息类型与完整流程](./52_DTI协议分层_消息类型与完整流程.md)
- [DTI连接管理与Token生命周期](./54_DTI连接管理与Token生命周期.md)
- [BAS互连组件与TID_TDEST路由](./55_BAS互连组件与TID_TDEST路由.md)
- [DTI失效与同步_INV_ACK和SYNC_ACK](./57_DTI失效与同步_INV_ACK和SYNC_ACK.md)

原问答标识：`bbb21abe-c9a7-4a18-a66f-611319f6e48f`、`83fd2301-5f97-48e2-83be-a0ab1f61c3df`。
