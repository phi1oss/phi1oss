# SMMUv3 Stream Table 与 CPU 页表上下文

- 收录时间：2026-09-01
- 状态：Inbox / 待后续归并
- 主题：SMMUv3 Programming Interface、Registers、Root Control Page、Stream Table、STRTAB_BASE、CPU TTBR0_EL1 对比
- 关键词：SMMUv3, StreamID, Stream Table, STE, CD, SMMU_STRTAB_BASE, TTBR0_EL1, ASID, VMID, Translation Context, RME, Root Control Page

## 1. SMMUv3 Programming Interface

SMMUv3 的编程模型可以概括为：**少量寄存器 + 大量 memory-based configuration structures + Command/Event Queue**。

- SMMU Registers：负责 capability discovery、全局 enable/control、interrupt/error，以及告诉硬件 Stream Table、Command Queue、Event Queue 等结构的 base address/size。
- Stream Table / STE / CD：保存 per-Stream / per-context 的 translation 配置。
- Translation Table：保存实际的地址映射。
- Command Queue：Software → SMMU 的异步维护命令通道，例如 TLBI、configuration invalidation、SYNC。
- Event Queue：SMMU → Software 的 fault/event 通知通道。

一句话：**STE/CD 决定怎么翻，Page Table 决定翻到哪，CMDQ 决定软件让 SMMU 做什么维护，EVTQ 决定 SMMU 告诉软件发生了什么。**

## 2. 为什么 SMMU 维护使用 Command Queue

CPU 的 TLBI 系统指令首先面向 CPU 自己的 translation state；SMMU 是独立 translation agent，维护自己的 Stage 1/Stage 2 TLB、STE/CD configuration cache，以及大量 Stream/ASID/VMID 相关状态。

因此软件不是“替 SMMU 清 TLB”，而是通过 Command Queue 告诉 SMMU：**请你维护你自己的 translation/configuration state。**

Command Queue 的优势：

- 可以排队；
- 可以批量提交；
- 可以异步执行；
- 能表达 SMMU 专用操作，而不只是 VA-based TLBI；
- 通过 CMD_SYNC 确认此前命令真正完成。

补充：支持 Broadcast TLB Maintenance 时，某些 CPU TLBI 可以通过 DVM 广播影响 SMMU，但不能替代 Command Queue 这一完整管理接口。

## 3. SMMU Registers

SMMUv3 的寄存器主要承担“入口和总控”，而不是为每个 Stream 保存整套 translation context。

典型分类：

- ID registers：能力发现；
- Control registers：SMMU / Queue 等全局控制；
- Interrupt registers：事件通知配置；
- Global error registers：全局/编程错误；
- Stream Table registers：配置 Stream Table base / format / size；
- Command Queue registers：配置 CMDQ；
- Event Queue registers：配置 EVTQ；
- Address translation registers：可选软件 translation/query 类接口；
- Implementation-defined registers：实现特定扩展。

TrustZone 模式下，Secure 与 Non-secure 有独立寄存器视图。

## 4. RME Root Control Page

实现 RME 时，会增加 Root PAS 专用的 Root Control Page。

核心用途：

- GPT configuration；
- GPT TLBI / maintenance；
- GPT fault registers；
- Root-level control；
- Root capability / ID registers。

它只能由 Root PAS 访问，其他 PAS 访问按 RAZ/WI 处理。

核心理解：**普通 SMMU translation 主要解决“地址翻到哪里”，而 GPT/GPC 进一步解决“这个物理 granule 属于哪个 PAS、当前访问者是否允许访问”。**

## 5. Stream Table 与 STE

Stream Table 是 SMMUv3 放在系统内存中的 per-Stream 配置表。Incoming transaction 的 StreamID 用来选择对应 STE。

逻辑关系：

```text
StreamID
   ↓
Stream Table
   ↓
STE
   ├─ V：Stream 配置是否有效
   ├─ Config：Bypass / Stage1 / Stage2 / S1+S2
   ├─ Stage 2：VMID、S2TTB 等
   └─ S1ContextPtr
          ↓
       CD / CD Table
          ↓
   Stage 1 Translation Context
```

一句话：**SID → STE；STE 管 Stream 和 Stage 2；SSID → CD；CD 管 Stage 1。**

## 6. SMMU_STRTAB_BASE 是什么

`SMMU_STRTAB_BASE` 保存的是**整个 Stream Table 在系统物理内存中的基地址**，由 SMMU driver / OS 在初始化阶段分配内存后写入。

它不是：

- 某个 VM 的 Stage 2 page-table base；
- 某个进程的 Stage 1 page-table base；
- 每个进程各自一份的寄存器。

层级可以这样区分：

```text
SMMU_STRTAB_BASE
→ 整张 Stream Table 在哪里

StreamID
→ 选择 STE

STE.S2TTB
→ 某个 Stream / VM 的 Stage 2 页表入口

SSID
→ 选择 CD

CD.TTB0
→ 某个 Substream / 进程的 Stage 1 页表入口
```

因此多个 VM / Device 可以共享一张 Stream Table，但使用不同的 STE、VMID、S2TTB 或 CD。

## 7. 与 CPU TTBR0_EL1 的对比

CPU 侧多个用户进程通常不会共用一个固定的用户页表根地址。

每个进程拥有自己的 Stage 1 用户页表；每个 PE 上的 `TTBR0_EL1` 表示**当前 active 用户地址空间的页表入口**。进程 context switch 时，OS 通常会把新进程的页表根地址和 ASID 切换到当前 PE 的 `TTBR0_EL1`。

例如：

```text
Process A
PageTable_A + ASID 1
        ↓
TTBR0_EL1（运行 A 时）

context switch
        ↓

Process B
PageTable_B + ASID 2
        ↓
TTBR0_EL1（运行 B 时）
```

TLB 可以借助 ASID 同时缓存不同进程相同 VA 的 translation，而不会混淆。

### CPU 与 SMMU 的本质差异

```text
CPU：
Scheduler 切换当前进程
→ 切换 TTBR0_EL1 / ASID
→ 一个 PE 当前选择一个 active user translation context

SMMU：
同时面对多个 Device / Process transaction
→ SID / SSID 选择 STE / CD
→ 多套 translation context 可以同时存在并被并发选择
```

一句话：**CPU 靠“当前执行上下文 + TTBR0_EL1”选择页表；SMMU 靠“SID/SSID + STE/CD”选择 translation context。**

## 8. 当前知识连接

这组内容应与以下主题后续归并：

- SMMUv3 Programming Model
- StreamID / SubstreamID
- Stream Table / STE / Context Descriptor
- Stage 1 / Stage 2 Translation
- Command Queue / Event Queue
- RME / GPT / GPC
- CPU MMU 与 SMMU Translation Context 对比

## 9. 待确认 / 后续展开

- Stream Table 的 linear / 2-level format 以及 StreamID 到 STE 地址的精确计算；
- STE.Config / STRW 的完整架构编码与使用场景；
- CD 中与 CPU TCR/TTBR/MAIR 对应字段的逐项比较；
- 修改 STE/CD 后，Configuration Cache invalidation 的完整软件顺序；
- 多核 CPU context switch 时 TTBR0_EL1、ASID、TLBI 的实际 OS 流程。

## 10. 面试记忆

> **SMMU_STRTAB_BASE 找整张 Stream 配置表；SID 找 STE；SSID 找 CD；STE/CD 决定 translation context；真正的 Stage 1 / Stage 2 页表基地址分别由 CD / STE 中相应字段给出。CPU 则通过当前 PE 的 TTBR0_EL1 + ASID 选择当前用户进程的 Stage 1 translation context。**
