# DTI 连接管理与 Token 生命周期

记录日期：2026-09-08
记录端：本地

## 来源与整理范围

来源：[SMMU 对话](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)，2026-09-08 的相关问答。

本篇为 Inbox 来源记录（Capture），不是已查证晋升的正式知识。问题按原意凝练；回答保留原文，仅整理标题层级、排版及移除不可解析的引用标记。原回答中的“Slide 明确写”“TRM 明确说明”等为原答表述，本次未重新打开规范逐条复核。截图图像未随本篇保存，可回原会话查看；截图若来自培训视频，则为“培训视频截图，时间点未知”，原会话未提供视频时间戳。

## Connection / Disconnection

### 问题

DTI 的连接与断开如何进行？manager/subordinate 是什么角色，为什么要在连接阶段协商 Token、断连时归还资源？

### 回答

这页讲的是 **DTI 链路的连接/断开握手**。核心可以理解为：

> **TBU / PCIe RC 在真正使用 DTI 发送 translation 等消息前，要先向 TCU 建立逻辑连接并协商 Token；准备掉电前，再主动断开并归还资源。**

##### 1. 谁发起？

Slide 里定义：

```text
DTI manager      = TBU / PCIe RC
DTI subordinate  = TCU
```

连接一定由 DTI manager 发起：

```text
TBU
 │ CONDIS_REQ(connect)
 ▼
TCU
 │ CONDIS_ACK(connect)
 ▼
TBU
```

TCU可以：

- 接受连接
- 或拒绝连接

所以不是“物理线接上就能直接发 DTI Translation Request”。

---

##### 2. 为什么还要有 connect？

因为 DTI 使用我们上一页讲的 **Token flow control**。

建立连接时，不只是：

> “你好，我们开始通信。”

更重要的是建立双方的 flow-control resource：

```text
TBU                   TCU
 │                     │
 │  请求 translation   │
 │  token              │
 │                     │
 │  提供 invalidation  │
 │  token              │
 └────────↔────────────┘
```

其中：

- **Translation Token**：TCU 提供，限制 TBU → TCU 的 outstanding translation requests。
- **Invalidation Token**：TBU / PCIe RC 提供，限制 TCU → TBU 的 outstanding invalidations。

所以：

> **Connect 本质上也是一次 DTI 通信资源初始化。**

---

##### 3. 状态怎么走？

右边图最重要：

```text
DISCONNECTED
     │
     │ CONDIS_REQ(connect)
     ▼
REQ_CONNECT
     │
     │ CONDIS_ACK(connect)
     ▼
CONNECTED
```

进入 `CONNECTED` 后，才能正常进行：

```text
TRANS_REQ
TRANS_RESP
INV
SYNC
...
```

之后准备关闭：

```text
CONNECTED
     │
     │ CONDIS_REQ(disconnect)
     ▼
REQ_DISCONNECT
     │
     │ CONDIS_ACK(disconnect)
     ▼
DISCONNECTED
```

可以把它理解成一个很简单的状态机：

```text
DISCONNECTED
     ↓ connect
CONNECTED
     ↓ disconnect
DISCONNECTED
```

中间的：

```text
REQ_CONNECT
REQ_DISCONNECT
```

就是正在等待对端确认的 transient state。

---

##### 4. 为什么 reset 后要 Connect？

Slide：

> `Connect after reset`

因为 reset 后原来的 DTI connection/resource 状态不能继续假设有效：

```text
Reset
 ↓
TBU = DISCONNECTED
 ↓
重新 Connect
 ↓
重新建立 Token/resource
 ↓
开始 translation service
```

---

##### 5. 为什么 power-down 前必须 Disconnect？

这个和你以前学 PPU/低功耗的思路很像。

如果 TBU 直接掉电：

```text
TBU
 ↓ power off

TCU却还认为：
“这个TBU已经CONNECTED，
而且我还给它分配了translation tokens”
```

显然资源状态会不一致。

所以要先：

```text
准备power down
     ↓
CONDIS_REQ(disconnect)
     ↓
归还/释放connection资源
     ↓
TCU确认
     ↓
DISCONNECTED
     ↓
再power down
```

Slide明确写：

> **At disconnection, returns translation tokens to TCU**

所以 Disconnect 很像：

> **“我要下线了，把之前占用的 DTI 资源一起退掉。”**

---

##### 6. 图里的 `N`、`M` 怎么理解？

你看到：

```text
CONDIS_REQ(connect, N)

CONDIS_ACK(connect, M)
```

它们和连接时的 Token/resource negotiation 有关。

这张 slide 本身没有展开 `N/M` 的精确定义，所以现在不用死记具体字段编码；先理解：

> **连接握手除了 connect/disconnect 类型，还会交换双方需要的 flow-control resource 信息。**

后面如果培训文档专门展开 `CONDIS_REQ/ACK` message format，我们再精确到每个 field。

---

#### 和上一页 Token 串起来

你可以把整个 DTI 启动过程记成：

```text
TBU reset完成
    ↓
DISCONNECTED
    ↓
CONDIS_REQ
    ↓
协商 Translation / Invalidation Token
    ↓
CONDIS_ACK
    ↓
CONNECTED
    ↓
正常 TRANS_REQ / RESP
    ↓
准备power down
    ↓
CONDIS_REQ(disconnect)
    ↓
归还Token / 释放connection
    ↓
CONDIS_ACK
    ↓
DISCONNECTED
```

##### 一句话记忆

> **DTI Connect = “先建立关系、分配 Token，再开始工作”；DTI Disconnect = “先归还资源、结束关系，再允许掉电”。**

## 一句话总结

DTI connection 不只是连通线路，还建立协议状态并分配资源；断连需要按协议完成相关事务收尾和资源回收。

## 整理注与待核实边界

截图中的 N、M 没有在原回答中落实到精确字段定义；保留这一边界，不自行补全编码或断电时序。

## 关联记录

- [DTI协议概览与Token流控](./53_DTI协议概览与Token流控.md)
- [DTI远程访问TBU寄存器](./58_DTI远程访问TBU寄存器.md)

原问答标识：`236e75c5-8cea-4c31-941f-4dffbcd81a93`。
