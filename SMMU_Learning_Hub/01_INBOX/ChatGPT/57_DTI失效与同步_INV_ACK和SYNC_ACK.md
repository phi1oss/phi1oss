# DTI 失效与同步：INV_ACK 和 SYNC_ACK 的边界

记录日期：2026-09-08
记录端：本地

## 来源与整理范围

来源：[SMMU 对话](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)，2026-09-08 的相关问答。

本篇为 Inbox 来源记录（Capture），不是已查证晋升的正式知识。问题按原意凝练；回答保留原文，仅整理标题层级、排版及移除不可解析的引用标记。原回答中的“Slide 明确写”“TRM 明确说明”等为原答表述，本次未重新打开规范逐条复核。截图图像未随本篇保存，可回原会话查看；截图若来自培训视频，则为“培训视频截图，时间点未知”，原会话未提供视频时间戳。

## Invalidation / Synchronization

### 问题

DTI 的 INV_REQ、INV_ACK、SYNC_REQ 和 SYNC_ACK 分别代表什么？为什么收到 INV_ACK 不能认为失效已完成？

### 回答

这页最关键的是分清：

> **`INV_ACK` 只是“我收到了，token 可以还你”；`SYNC_ACK` 才是“前面的 invalidation 真的都做完了”。**

##### 1. `INV_REQ`：要求失效缓存

TCU 向 TBU / PCIe RC 发送：

```text
DTI_<TBU/ATS>_INV_REQ
```

作用：

> **要求对 TLB 或相关 configuration cache entry 做 invalidation。**

例如：

```text
CPU修改页表
   ↓
CMDQ: TLBI
   ↓
TCU
   ↓ INV_REQ
TBU
   ↓
清理匹配的TLB entry
```

请求里可以带：

```text
VA
ASID
SID
type
...
```

用于说明“清哪些 entry”。

---

##### 2. `INV_ACK`：不是完成确认

这是这页最容易误解的地方。

Slide 明确写：

> `Simply an ack for flow control purposes`

也就是说：

```text
TCU → INV_REQ → TBU
                 ↓
              INV_ACK
                 ↓
TCU
```

`INV_ACK` 的意义主要是：

> **这笔 invalidation request 已经被 TBU 接收，因此对应的 Invalidation Token 可以归还。**

但此时：

```text
INV_ACK
≠ invalidation effects 全部完成
```

TBU 内部可能仍然有：

- 正在处理的相关 transaction
- invalidation pipeline
- outstanding translation effects

---

##### 3. `SYNC_REQ`：问“前面的 invalidation 真做完了吗？”

所以需要第二步：

```text
TCU → SYNC_REQ → TBU
```

它不是再清一次 TLB，而是在问：

> **“在这个 SYNC 之前发送的 invalidation，现在是否真正完成？”**

---

##### 4. `SYNC_ACK`：真正的 completion point

TBU 必须等到：

```text
所有之前相关 invalidation
+
需要等待的architectural effects
```

都真正结束后，才能：

```text
TBU → SYNC_ACK → TCU
```

所以最适合你的理解方式是：

> **前面的 invalidation 没有真正收尾之前，`SYNC_ACK` 不能回来。**

---

#### 整个流程

```text
TCU                           TBU

INV_REQ #1  ────────────────>
INV_REQ #2  ────────────────>

             <──────── INV_ACK
             <──────── INV_ACK
             // 这里只说明请求被接收，
             // token已经释放

SYNC_REQ    ────────────────>
                   ↓
          等前面INV真正完成
                   ↓
             <──────── SYNC_ACK
```

因此：

```text
INV_ACK
→ flow-control completion

SYNC_ACK
→ architectural invalidation completion
```

---

#### 为什么要分成两级？

因为如果每个 `INV_REQ` 都必须等 invalidation 完全结束才能还 token：

```text
INV1 → 等完成 → ACK
INV2 → 等完成 → ACK
INV3 → ...
```

并行度会很差。

现在可以：

```text
INV1
INV2
INV3
INV4
 ↓
快速ACK / 回收token
 ↓
并行处理
 ↓
最后发一个SYNC
 ↓
统一确认全部完成
```

这和前面 CMDQ 的思想非常像：

```text
CMD_TLBI / CMD_CFGI
→ 异步发起维护

CMD_SYNC
→ 最后确认前面的维护真正完成
```

DTI 层对应：

```text
INV_REQ
→ 发起失效

SYNC_REQ / SYNC_ACK
→ 确认失效真正完成
```

---

##### Slide 里说什么时候会触发？

主要三类：

```text
1. TCU收到 DVM TLBI / Sync
   → 向TBU传播invalidate/sync

2. TCU从CMDQ读到 invalidation / sync command
   → 向相关TBU发送DTI消息

3. 某些TCU寄存器变化
   → implicit invalidation
```

其中 slide 特别注明：

> DVM TLBI / Sync 不是用来做 configuration cache invalidation 的。

---

##### 一句话记忆

> **INV_REQ 是“去清缓存”，INV_ACK 是“请求我收到了”；SYNC_REQ 是“前面的都做完了吗”，SYNC_ACK 才是“真的全部做完了”。**

## 一句话总结

INV_ACK 是流控确认；SYNC_ACK 才确认相应同步范围内的失效及其要求的效果完成。

## 整理注与待核实边界

原回答将 TBU/ATS 的消息作概括介绍；各自允许的失效对象、字段和同步范围仍需分别对照协议核实。关于提升并行度的解释属于设计动机推导，不等同于规范规定的具体微架构。

## 关联记录

- [DTI协议概览与Token流控](./53_DTI协议概览与Token流控.md)
- [CMD_SYNC的本质与完成边界](./33_CMD_SYNC的本质与完成边界.md)
- [CFGI_TLBI与PTE更新边界](./31_CFGI_TLBI与PTE更新边界.md)

原问答标识：`2e54e810-f07c-47e5-8ca0-4ea727edfa88`。
