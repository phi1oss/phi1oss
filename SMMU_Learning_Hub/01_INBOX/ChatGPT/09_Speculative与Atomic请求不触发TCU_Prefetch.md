# 为什么 Speculative Translation Request 和 Atomic Transaction 不触发 TCU Prefetch

## 问题

即使 TCU prefetch 已启用，为什么下面两类 original translation request 也不会触发 prefetch？

```text
Speculative Translation Request
DTI_TBU_TRANS_REQ.PERM[1:0] = 2'b11

Atomic Transaction with Data Response
DTI_TBU_TRANS_REQ.PERM[1:0] = 2'b10
```

对应的原始 transaction 包括：

```text
Speculative：
StashOnceShared
StashOnceUnique
StashTranslation

Atomic with Data Response：
AtomicLoad
AtomicSwap
AtomicCompare
```

## 核心结论

**Speculative translation request 和 atomic transaction 不触发后续 TCU prefetch，本质上是因为它们不适合作为“下一段线性地址访问”的可靠预测依据。**

- Speculative request 本身已经是预测行为；如果继续派生 prefetch，可能形成 speculation chaining。
- Atomic transaction 通常针对特定的共享状态或同步地址，不体现稳定的线性地址局部性，因此预取下一段 translation 的收益较低。

> **TCU prefetch 希望由真实且具有线性访问特征的 demand translation 触发；speculative request 已经是预测，atomic request 通常又不具备线性趋势，所以二者都不会成为 prefetch trigger。**

## 1. TRM 明确规定了什么？

即使对应 STE 的 prefetch 控制 `STE.PF` 已经启用，TCU 也不会让所有 translation request 都触发下一笔 prefetch。

TRM 明确排除两类 original request：

```text
Speculative Translation Request
        ↓
不触发 TCU Prefetch

AtomicLoad / AtomicSwap / AtomicCompare
        ↓
带 Data Response 的 Atomic Transaction
        ↓
不触发 TCU Prefetch
```

因此，`STE.PF = Enabled` 只是允许使用 prefetch，并不意味着每种请求类型都具备触发资格。

## 2. Speculative Request：避免预测继续派生预测

正常的 prefetch 起点是真实的 demand translation：

```text
真实 Demand Translation
          ↓
TCU 完成 Translation
          ↓
根据当前结果预测下一段 Translation
          ↓
生成 Speculative Prefetch
```

但 speculative translation request 本身已经不是直接由真实访问需求驱动的普通请求。如果允许它再次触发 prefetch，可能形成：

```text
Speculative Request A
          ↓
      Prefetch B
          ↓
B 本身也是 Speculative
          ↓
      Prefetch C
          ↓
      Prefetch D
          ↓
          ……
```

这种 speculation chaining 可能在没有真实 requester demand 的情况下持续向前扩张，并消耗：

- Translation slot
- Walk Cache 容量
- Table-walk memory bandwidth
- 其他 TCU 内部处理资源

还可能把真正有用的内容挤出 cache，造成 cache pollution。

因此可以将设计思想概括为：

> **Speculation 不再继续派生新的 speculation。**

需要注意证据边界：

- **TRM 明确事实：**speculative original request 不触发 prefetch。
- **合理的微架构解释：**这样可以避免 speculation chaining、资源浪费和 cache pollution；TRM 原文没有直接声明这是唯一原因。

## 3. Atomic Transaction：缺少可靠的线性地址局部性

这里涉及的是带 data response 的 atomic transaction：

```text
AtomicLoad
AtomicSwap
AtomicCompare
```

TCU prefetch 最适合的访问模式通常是：

```text
Address Range A
      ↓
Address Range B
      ↓
Address Range C
      ↓
Address Range D
```

也就是连续、方向明确且重复出现的线性内存访问。

Atomic transaction 的访问模式通常不同：

```text
访问某个共享状态地址
          ↓
读取 / 比较 / 修改
          ↓
返回 Data Response
```

典型用途包括：

```text
Lock
Counter
Reference Count
Synchronization State
共享状态变量
```

这类 transaction 往往反复访问某个特定地址，不一定意味着随后会继续访问下一个 translation range。因此，以它作为“下一段 translation”的预测起点，命中收益通常有限，反而可能浪费 translation 和 table-walk 资源。

同样需要区分证据层次：

- **TRM 明确事实：**由 AtomicLoad、AtomicSwap、AtomicCompare 引起的这类 translation request 不触发 prefetch。
- **合理的系统解释：**atomic 地址模式通常缺少稳定的线性局部性，因此不适合作为预取依据；TRM 这一段没有直接把 locality 写成原因。

## 4. TCU Prefetch 的触发筛选

完整判断关系可以简化为：

```text
TCU 收到 Translation Request
             ↓
完成 Original Translation
             ↓
检查 STE.PF 是否 Enabled
             ↓
判断 Original Request Type
             │
      ┌──────┼───────────────┐
      │      │               │
   Normal  Speculative   Atomic with
   Demand   Request      Data Response
      │      │               │
      ▼      ▼               ▼
可考虑    不触发           不触发
Prefetch  Prefetch         Prefetch
```

这说明 TCU prefetch 并不是：

```text
任何 Translation 完成
          ↓
都自动预取下一段
```

而是：

```text
Translation 完成
          ↓
检查配置与 Request 类型
          ↓
只有合适的 Demand Translation
才可能触发 Prefetch
```

## 5. PERM 字段的额外含义

从这段 TRM 可以看到，`DTI_TBU_TRANS_REQ.PERM[1:0]` 不应简单理解成传统意义上的 Read/Write permission bits。

它的特定编码还能帮助 TCU 区分请求属性：

```text
PERM[1:0] = 2'b11
→ Speculative Translation Request

PERM[1:0] = 2'b10
→ Atomic Transaction with Data Response
```

TCU 根据这些属性判断该 translation request 是否适合作为 prefetch trigger。

## 对比总结

| Original Request | 特点 | 不触发 Prefetch 的设计理解 |
|---|---|---|
| Speculative Translation Request | 本身已经是预测请求 | 避免 speculation 继续派生 speculation |
| Atomic Transaction with Data Response | 常用于锁、计数器和共享状态 | 地址模式通常不体现可靠的线性局部性 |
| Normal Demand Translation | 由真实访问触发 | 在配置和其他条件满足时，可以用于预测下一段 translation |

## 一句话总结

> **Speculative request 已经是预测，不应继续派生预测；AtomicLoad/Swap/Compare 通常访问特定同步地址，不适合预测下一段线性 translation，因此二者都不会触发 TCU prefetch。**
