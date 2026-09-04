---
received_at: 2026-09-04 15:51 +08:00
source: SMMU聊天框
status: inbox
topic: PCIe bifurcation
---

# PCIe bifurcation 概念

## 核心定义

**PCIe bifurcation 是把一组较宽的 PCIe Lane 拆分成多个彼此独立的较窄 PCIe Link / Root Port。**

例如：

```text
x16
↓ bifurcation
x8 + x8
```

或者：

```text
x16
↓
x4 + x4 + x4 + x4
```

## 本质变化

它不只是“单条链路带宽变小”，而是：

> **一个宽 PCIe 接口被拆成多个独立的 transaction source / Root Port。**

因此 bifurcation 后，不同 Root Port 可以分别连接不同 Endpoint，并独立发起 PCIe transaction。

## 对 SMMU/MMU S3 集成的影响

问题会从：

```text
一个 PCIe Port
  ↓
一个 translation/data path
```

变成：

```text
多个 Root Port
  ↓
如何分别接入 SMMU translation path？
```

在培训文档第 66 页展示的两种思路中：

```text
ACE-Lite TBU
→ 多个数据通路需要相应的 inline TBU path

LTI TBU
→ 多个 Port 的 translation request
  可以集中送给 multi-port LTI TBU
  实际数据 transaction 直接进入 NOC
```

## 一句话记忆

> **PCIe bifurcation = 把一个宽 PCIe Port 拆成多个独立窄 Port；对 SMMU 的影响是 translation/data path 也要适配这些独立 Port。**
