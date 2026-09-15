# TBU Hazarding：合并重复的 Translation Miss

## 问题

如何理解下面这段 TRM 对 TBU Hazarding 的描述？

> If the MicroTLB lookup results in a miss, the transaction checks whether there are any pending transactions from which it can use the translation. This method is called forming a hazard. If the transaction hazards on any pending transactions, then the transaction waits until the response is available for the hazarded transaction and uses that response.

## 核心结论

**TBU Hazarding 本质上是一种合并重复 translation miss 的优化：如果前面已经有 pending transaction 正在请求相同的 translation，新 transaction 就不再重复请求 TCU，而是等待并复用前一笔 transaction 的 translation response。**

## 1. Forming a Hazard 是建立 Translation 依赖

假设两笔 transaction 访问同一个 translation granule：

```text
Transaction A
IOVA = 0x1234_1000
      │
      ▼
MicroTLB Miss
      │
      ▼
需要 Translation X
      │
      └──────────────► 向 TCU 发出 Translation Request
                         当前正在等待 Response

随后 Transaction B 到来
IOVA = 0x1234_1800
      │
      ▼
MicroTLB Miss
      │
      ▼
发现需要的也是 Translation X
      │
      ▼
不再发送第二个 Request
      │
      ▼
Hazard on Transaction A
      │
      ▼
等待 A 的 Translation Response
      │
      ▼
Response 返回
      │
      ├── A 使用该 Translation
      └── B 复用该 Translation
```

这里的 hazard 并不是发生了错误或资源冲突，而是：

> **TBU 检测到新 transaction 与某个 pending transaction 需要相同的 translation，于是建立依赖，让新 transaction 等待并复用前者的结果。**

## 2. 为什么需要 Hazarding？

假设 DMA Engine 连续访问同一个 4KB page：

```text
0x1000
0x1040
0x1080
0x10C0
```

如果这个 page 的 translation 尚未进入 TLB，而系统没有 hazarding，可能出现：

```text
Tx0 MicroTLB Miss → TCU Request
Tx1 MicroTLB Miss → TCU Request
Tx2 MicroTLB Miss → TCU Request
Tx3 MicroTLB Miss → TCU Request
```

四个 request 实际上都在查询同一个映射：

```text
同一个 IOVA Page → 同一个 PA Page
```

这些重复请求会浪费：

- TBU translation slot
- DTI bandwidth
- TCU Translation Manager slot
- Page-table walk bandwidth

启用 hazarding 后：

```text
Tx0
 │
 └── Translation Request → TCU

Tx1 ─┐
Tx2 ─┼── Hazard on Tx0
Tx3 ─┘
       │
       ▼
等待 Tx0 的 Translation Response
       │
       ▼
所有相关 Transaction 复用结果
```

因此，多个 transaction 需要同一个 translation 时，只需要向 TCU 发送一个 request。

## 3. 什么是 Pending Transaction？

前一笔 transaction 已经发生 MicroTLB miss，但 translation 尚未返回：

```text
MicroTLB Miss
      ↓
没有立即得到 Translation
      ↓
进入 Translation Manager
      ↓
正在查询 Main TLB
或者
已经通过 DTI 请求 TCU
      ↓
Translation Response 尚未返回
```

此时，它就是 pending transaction。

新 transaction 在 MicroTLB miss 后，会先检查：

> **“是否已经有其他 transaction 正在替我获取相同的 translation？”**

如果有，就建立 hazard dependency，而不是重复走完整的 miss path。

## 4. Hazarding 与 TLB Hit 的区别

Hazarding 不是一种特殊的 TLB hit：

```text
普通 TLB Hit

Transaction
    ↓
MicroTLB Hit
    ↓
立即得到 Translation
    ↓
继续执行
```

```text
Hazarding

Transaction
    ↓
MicroTLB Miss
    ↓
发现另一个 Pending Transaction
正在获取相同的 Translation
    ↓
等待
    ↓
复用它未来返回的 Response
```

二者的核心区别是：

```text
TLB Hit：
Translation 已经存在于 Cache 中

Hazarding：
Translation 尚未存在于 Cache 中，
但已经有其他 Transaction 正在获取它
```

## 5. 与 CPU Cache Miss Coalescing 的类比

TBU Hazarding 类似 CPU Cache 中的 miss coalescing 或 MSHR merge：

```text
多个 Request 访问同一个 Missing Cache Line
                 ↓
只发送一个 Memory Request
                 ↓
其他 Request 等待并共享返回结果
```

区别在于，CPU Cache 合并后共享的是 cache-line data，而 TBU Hazarding 共享的是：

```text
Translation Result
```

能够同时形成 hazard 的 unique address 数量是有限的，由 `TBUCFG_HZRD_ENTRIES` 参数控制。这意味着 TBU 内部需要维护 hazard tracking state，记录：

```text
哪些 Translation 正在 Pending
哪些后续 Transaction 正在等待它们
```

## 一句话总结

> **TBU Hazarding 是指多个 transaction 同时需要同一个尚未返回的 translation 时，只让其中一个真正发起 translation request，其余 transaction 等待并复用它的 response，从而避免 duplicate translation requests。**
