# AMBA Atomic Transaction

- 收录时间：2026-09-07
- 主题：AMBA / AXI5 Atomic Transaction
- 建议后续归档：协议与接口 / AMBA / Atomic Transaction
- 关键词：AMBA、AXI5、Atomic Transaction、AWATOP、Read-Modify-Write、Exclusive Access、MMU-720AE、TBU

## 一句话定义

AMBA Atomic Transaction 是一种由下游硬件一次性完成 **Read-Modify-Write** 的原子操作，Master 通过 `AWATOP` 指定要执行的原子运算。

## 为什么需要

它主要解决多个 Master 同时修改同一个共享地址时的竞争问题。

普通 Read + Write：

```text
Master A: Read 10
Master B: Read 10
A: Write 11
B: Write 11

最终 = 11   // 错，正确应为12
```

Atomic：

```text
Master A:
Atomic ADD 1
    ↓
Target执行：
old = 10
new = 11
write 11

Master B:
Atomic ADD 1
    ↓
old = 11
new = 12
write 12
```

## 核心信号与语义

- `AWATOP`：指定 Atomic operation 类型。
- `WDATA`：通常携带 operation 的 operand。
- 下游支持 Atomic 的组件负责不可分割地完成 Read-Modify-Write。
- 某些 Atomic operation 还会把修改前的旧值返回给 Master。

## 与 Exclusive Access 的区别

```text
Exclusive
= Master自己 Read → Modify → Write
  系统检查期间有没有被别人修改

Atomic
= Master直接说“帮我 ADD / SWAP / COMPARE”
  下游一次完成
```

因此：

- Exclusive：requester-side retry model。
- Atomic：subordinate-side atomic operation。

## 在 MMU-720AE / TBU 中的理解

```text
Atomic request + IOVA
        ↓
       TBU
        ↓
IOVA → PA
Atomic语义保持不变
        ↓
System
```

TBU 本身不是执行 Atomic 运算的 ALU；它的职责是完成地址翻译，同时保证 Atomic transaction 的协议语义在穿过 TBU 后仍然保持正确。

## 一句话记忆

> **Exclusive 是“我自己读改写，你帮我看有没有冲突”；Atomic 是“我告诉你怎么改，你一次性替我完成”。**

## 待进一步确认/扩展

- AXI5 `AWATOP` 的具体编码分类。
- AtomicLoad / AtomicStore / AtomicSwap / AtomicCompare 的返回通道行为。
- Atomic transaction 与 ACE coherency、cache line ownership 的具体交互。
