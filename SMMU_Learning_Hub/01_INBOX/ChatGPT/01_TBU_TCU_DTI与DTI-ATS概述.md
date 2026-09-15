# TBU、TCU、DTI Interconnect 与 DTI-ATS 概述

## 问题

在面试场景中，如何简单凝练地概述 TBU、TCU、DTI Interconnect 和 DTI-ATS？

## 回答

**TBU（Translation Buffer Unit）**：靠近设备侧的地址翻译前端，主要负责查本地 TLB。命中时直接完成翻译；未命中时通过 DTI 向 TCU 请求 translation。

**TCU（Translation Control Unit）**：SMMU 的中央翻译控制单元，负责更复杂的工作，比如 configuration lookup、page table walk、GPT walk，以及统一管理 translation 相关状态。

**DTI Interconnect**：连接多个 TBU/ATS requester 与 TCU 的 translation/control 网络。它负责承载 DTI message，本身不做地址翻译，典型组件包括 switch、sizer、register slice。

**DTI-ATS**：PCIe ATS 场景下，PCIe Root Port 与 TCU 之间的 DTI 通路，用来把 PCIe Endpoint 的 ATS translation request 直接送到 TCU，并返回 translation result；它传的是 translation/control 信息，不是 PCIe DMA payload。

## 一句话总结

> **TBU 负责“本地快速翻译”，TCU 负责“中央复杂翻译”，DTI Interconnect 负责“把它们连起来”，DTI-ATS 则让 PCIe ATS 设备也能直接使用 TCU 的 translation service。**
