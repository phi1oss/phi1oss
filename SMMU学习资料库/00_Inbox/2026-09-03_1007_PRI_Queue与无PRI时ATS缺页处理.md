---
received_at: 2026-09-03 10:07 +08:00
source: SMMU聊天框
status: inbox
topic: PRI Queue / ATS Page Fault Recovery
source_material: MMU.pdf 物理第51页（PRI Queue）
---

# PRI Queue 与无 PRI 时 ATS 缺页处理

> 本条按手机端 Inbox 方式记录：先保留原问题、核心结论与完整交互流程，不做过度归档；后续可再并入 `04_地址转换流程`，并与 `05_软件编程模型` 建立关联。

## 原问题 1：PRI Queue 是什么，ATS 缺页时怎么处理？

培训文档物理第 51 页介绍 PRI Queue：它是一个可选的、位于内存中的队列，用于保存通过 DTI-ATS 收到的 PRI Page Request。其结构与 Event Queue 类似。

### 核心结论

**PRI 是 ATS 的补充机制。ATS 负责“问这个地址怎么翻”，PRI 负责“现在因为 page not present 翻不了，请软件把这个页准备好”。**

### 典型流程

```text
PCIe Device
    │
    │ 1. ATS Translation Request
    ▼
Root Complex
    │
    │ DTI-ATS
    ▼
SMMU
    │
    │ 2. Translation: page not present
    ▼
ATS Translation Completion: failure
    │
    ▼
PCIe Device
    │
    │ 3. PRI Page Request
    ▼
Root Complex
    │ DTI-ATS
    ▼
SMMU
    │
    │ 4. 写 PRI Entry
    ▼
PRI Queue
    │
    │ SMMU_PRIQ_PROD++
    ▼
Software
    │
    │ 5. 分配/调入 page，建立或更新页表映射
    │ 6. 完成相关 translation maintenance
    │ 7. SMMU_PRIQ_CONS++
    │ 8. 通过 CMDQ 提交 CMD_PRI_RESP
    ▼
SMMU
    │
    ▼
Root Complex
    │
    ▼
PCIe Device
    │
    ├─ success → 重新进行 ATS / 后续访问
    └─ failure → 本次 Page Request 失败
```

### PROD / CONS 的角色

```text
PRIQ Producer = SMMU
PRIQ Consumer = Software
```

- `SMMU_PRIQ_PROD`：SMMU 写入新的 PRI Entry 后推进。
- `SMMU_PRIQ_CONS`：软件读取并处理 PRI Entry 后推进。

### CMD_PRI_RESP 的意义

软件把页准备好并更新 PRIQ_CONS，并不等于 PCIe Endpoint 已经知道处理结果。软件还需要通过 Command Queue 发 `CMD_PRI_RESP`，明确告诉 Endpoint：这次 PRI 请求成功还是失败。

因此：

```text
PRIQ            ：Device → SMMU → Software，提出“请准备这个 page”
CMD_PRI_RESP    ：Software → CMDQ → SMMU → Device，返回处理结果
```

---

## 原问题 2：如果 MMU/SMMU 不支持 PRI，ATS 遇到缺页怎么办？

### 核心结论

**没有 PRI 时，ATS 仍然可以返回 translation failure，但没有“Device → PRI → Software demand paging”的标准恢复通道。**

也就是说：

```text
ATS Request
   ↓
SMMU Translation
   ↓
Page not present
   ↓
ATS Completion: failure
   ↓
没有 PRI 时到此为止
```

SMMU 不会因为 ATS 缺页自动：

```text
生成 PRI Request
→ 写 PRIQ
→ 唤醒 Page Fault Handler
→ CMD_PRI_RESP
```

### 处理方式 1：预先把 Device 会访问的页面准备好

这是没有 PRI 时最直接的设计方式：

```text
Software / Driver
   ↓
提前分配或 fault-in page
   ↓
建立 SMMU translation mapping
   ↓
必要时 pin / 保持 resident
   ↓
完成 TLBI / SYNC 等维护
   ↓
启动 Device
   ↓
Device ATS Request
   ↓
Translation success
```

核心思想：**不能依赖“缺页以后再补”，而要尽量保证 ATS 查询时 mapping 已经存在。**

### 处理方式 2：改走普通 SMMU fault / stall recovery 路径

如果 Device 发的是普通 Untranslated DMA，而对应 Stream 支持 Stall Model，则可以出现：

```text
Device DMA
   ↓
SMMU Translation
   ↓
Translation Fault
   ↓
Stall transaction
   ↓
EVTQ → Software
   ↓
Software 修复页表
   ↓
Resume Command
   ↓
继续 transaction
```

这和 PRI 的目标类似，都是“fault 后由软件修复”，但机制不同：

```text
PRI
→ PCIe Device 主动发 Page Request
→ 面向 ATS / SVA 的 Page Fault Recovery

Stall Model
→ SMMU 自己把普通 transaction stall
→ 通过 EVTQ 报告 Software
```

**Stall 不是 PRI 的协议替代物**，只是某些系统中另一条 fault-recovery 路径。

---

## ATS、PRI、PRIQ、CMD_PRI_RESP 的关系

```text
ATS
→ “这个地址怎么翻？”

ATS Failure: page not present
→ “现在没法翻”

PRI
→ “请软件把这个 page 准备好”

PRIQ
→ SMMU 把 Device 的 Page Request 暂存在内存中交给 Software

CMD_PRI_RESP
→ Software 把最终 success/failure 通过 SMMU 返回给 Endpoint
```

---

## 和 SVA 的联系

SVA 场景下，Device 可以使用进程地址空间。如果支持 ATS + PRI，则可以形成 Device-side demand paging：

```text
Device ATS
   ↓
CPU/SMMU 共享的进程页表中 page not present
   ↓
PRI
   ↓
Software Page Fault Handler
   ↓
CPU Page Table 更新
   ↓
CMD_PRI_RESP
   ↓
ATS Retry
   ↓
Translation Success
```

如果没有 PRI，SVA 的“Device 触发缺页并让 OS 动态补页”能力会受限，通常需要软件提前把相关 user pages 准备好并保持可用。

---

## RTL / 集成视角

需要重点确认：

1. SMMU 是否实现 PRI feature / PRIQ。
2. PCIe Endpoint 与 Root Complex 是否支持 PRI。
3. DTI-ATS 路径是否支持 Page Request / Page Response 相关消息。
4. PRIQ 的 Base、PROD、CONS 和 IRQ 是否正确配置。
5. 软件是否真正消费 PRIQ 并提交 `CMD_PRI_RESP`，不能只推进 CONS。
6. 若系统不支持 PRI，需要明确 Device buffer 是否在 workload 启动前完成 mapping / residency 保证。

---

## 验证与调试关注点

建议至少覆盖：

```text
Case 1: ATS translation success
→ 不产生 PRI

Case 2: ATS page-not-present + PRI supported
→ ATS failure
→ PRI request
→ PRIQ entry
→ IRQ
→ Software 修页表
→ CMD_PRI_RESP success
→ ATS retry success

Case 3: PRI response failure
→ Endpoint 收到 failure，不能错误继续使用地址

Case 4: PRIQ full / Software 消费不及时
→ 检查 overflow / backpressure / error 行为

Case 5: PRI unsupported
→ ATS page-not-present 只返回 failure
→ 不应错误产生 PRIQ entry
```

---

## 易混淆点

- **PRI Queue 不保存原始 DMA data payload**，保存的是 Page Request Record / metadata。
- **ATS failure 不等于一定会有 PRI**；只有 Device、Root Complex、SMMU 及软件链路支持并启用 PRI 时，才能进入 PRI recovery。
- **PRIQ_CONS++ 不等于 Page Response 已经发回 Endpoint**；软件还需要 `CMD_PRI_RESP` 完成闭环。
- **Stall Model 与 PRI 不是同一个机制**：前者是 SMMU stall 普通 transaction，后者是 PCIe Device 主动 Page Request。

---

## 面试式总结

> PRI 是 PCIe ATS 的缺页恢复机制。当 ATS Translation Request 因 page not present 失败时，支持 PRI 的 Device 可以发送 Page Request，SMMU 将请求写入 PRIQ，Software 准备页面并更新映射后，再通过 CMDQ 发送 `CMD_PRI_RESP` 告诉 Endpoint 成功或失败。如果系统不支持 PRI，则 ATS 缺页只能返回 translation failure，无法通过标准 PRI 流程进行 demand paging，此时通常需要软件提前建立并保持相关 mapping，或者让访问改走普通 SMMU fault/stall recovery 路径。

### 一句话记忆

> **ATS 是“问地址怎么翻”；PRI 是“翻不了，请 OS 补页”；没有 PRI，就要提前把页准备好，或使用另一条 fault recovery 路径。**

---

## 待确认 / 后续可继续展开

- PCIe ATS Translation Completion 对 page-not-present failure 的精确编码。
- PCIe PRI Page Request Group / Page Request Record 的具体字段。
- DTI-ATS 中 Page Request / Page Response 的精确 message 字段与握手。
- MMU-720AE 中 PRIQ overflow、PRI IRQ 和错误恢复的具体寄存器流程。
