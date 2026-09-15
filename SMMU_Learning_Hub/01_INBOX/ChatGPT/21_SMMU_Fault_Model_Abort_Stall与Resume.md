# SMMU Fault Model：Abort、Stall 与 Resume

> 记录日期：2026-08-30  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 来源材料：培训截图，时间点未知  
> 状态：Inbox 问答整理，待结合 SMMUv3 规范查证命令和事件细节

## 凝练问题

**SMMU 发生 translation fault 后如何处理设备 transaction？Abort 与 Stall 有什么区别，软件怎样通过 Resume 或 Terminate 结束处理，Stall 又为什么能够支持 Device Demand Paging？**

## 核心结论

> **SMMU 的 Fault Model 可以针对不同 Stream 选择 Abort 或 Stall。Abort 模式下，faulting transaction 直接失败；Stall 模式下，SMMU 暂停 transaction 并向软件报告 fault，软件修复页表后让 SMMU Resume/retry，或者在确认访问非法后将其 Terminate/Abort。**

这组内容关注的是：

```text
Fault 已经发生
      ↓
这笔 Device Transaction 接下来怎么办？
```

而不是 fault 类型本身。

## 1. Fault 如何产生

设备发出 DMA 请求：

```text
DMA
 │ Read IOVA / IPA
 ▼
SMMU
```

SMMU 执行 translation 或保护检查时，可能发现：

```text
PTE Invalid
Permission 不允许
地址范围或配置错误
其他 Translation Fault
```

于是：

```text
DMA Request
    ↓
SMMU Translation / Check
    ↓
Fault
```

Fault 之后主要有 Abort 与 Stall 两种处理策略。

## 2. Abort 模式：立即结束 Transaction

```text
Translation Fault
       ↓
      Abort
       ↓
向 Upstream Master 返回错误
```

完整含义是：

```text
DMA Request
    ↓
SMMU Translation 失败
    ↓
Abort Response
    ↓
这笔 Transaction 结束
```

特点是：

> **SMMU 不等待软件建立或修复 mapping，这笔访问直接失败。**

这种模式可用于：

- 设备不支持等待或重试。
- 访问明显非法，不应动态修复。
- 软件不希望为该 Stream 提供 Demand Paging。

## 3. Stall 模式：暂时挂起 Transaction

```text
DMA Request
    ↓
SMMU
    ↓
Translation Fault
    ↓
Hold / Stall Transaction
    ↓
向软件报告 Fault
```

此时原始 DMA transaction：

- 没有继续访问 memory。
- 也没有立即失败结束。
- 由 SMMU 暂时保存或挂起，等待软件决定。

SMMU 通常通过 fault/event reporting 和中断机制通知 CPU/software。

## 4. 软件收到 Fault 后做什么

软件读取 SMMU 报告的信息，确定：

```text
哪个 Stream / Context？
哪个 Input Address？
哪一种 Fault？
Transaction 是 Read 还是 Write？
```

如果这是合法但尚未建立 mapping 的请求，软件可以更新 translation table：

```text
原来：
IOVA 0x4000 → Invalid

修复后：
IOVA 0x4000 → PA 0x80004000
```

实际处理还必须包括适用的页表写入可见性、configuration/TLB invalidation 和同步，不能只修改内存中的 PTE。

## 5. Resume：修复后重新尝试 Translation

页表或配置修复完成后，软件通知 SMMU 继续处理被暂停的 transaction：

```text
CPU / Software
      ↓
Resume Command / Action
      ↓
SMMU
```

随后：

```text
Stalled DMA
    ↓
Resume
    ↓
重新执行 Translation
    ↓
找到有效 Mapping
    ↓
IOVA / IPA → PA
    ↓
完成 Memory Access
```

完整路径是：

```text
DMA Request
    ↓
SMMU Fault
    ↓
Stall
    ↓
Fault Event / Interrupt
    ↓
CPU / Software
    ↓
Update Translation Table
    ↓
Invalidate / Synchronize as required
    ↓
Resume
    ↓
SMMU Retry Translation
    ↓
Success
```

## 6. Stall 为什么可以支持 Demand Paging

Demand Paging 的含义是：页面开始时可以没有 mapping，第一次真正访问时再由软件分配页面并建立映射。

CPU 的流程：

```text
CPU Access
   ↓
Page Fault
   ↓
OS 分配或调入 Page
   ↓
Update PTE
   ↓
Retry Instruction
```

SMMU Stall Model 是设备侧的对应机制：

```text
Device DMA
   ↓
SMMU Translation Fault
   ↓
Stall Transaction
   ↓
Software 建立 Mapping
   ↓
Resume
   ↓
Retry DMA Translation
```

可以类比为：

> **CPU Page Fault + Instruction Retry，对应 SMMU Stall Fault + Transaction Resume。**

## 7. GPU Demand Paging 示例

GPU Process 访问：

```text
IOVA = 0x4000
PTE  = Invalid
```

如果使用 Abort：

```text
GPU Request
   ↓
SMMU Fault
   ↓
Transaction 失败
```

如果使用 Stall：

```text
GPU Request
   ↓
SMMU 暂时挂起
   ↓
CPU 收到 Fault
   ↓
分配 Physical Page
PA = 0x90000000
   ↓
建立 IOVA 0x4000 → PA 0x90000000
   ↓
执行必要的维护和同步
   ↓
Resume
   ↓
GPU Request 最终完成
```

## 8. Terminate：确认非法后结束访问

Stall 后，软件不一定必须 Resume。如果检查发现这是越权或非法 DMA，可以选择终止：

```text
Stalled Transaction
      ↓
Terminate / Abort Action
      ↓
SMMU 结束 Transaction
      ↓
向 Upstream Master 返回失败
```

所以 Stall 后有两种结局：

```text
                     Fault
                       ↓
                     Stall
                       ↓
                  Software 判断
                   /           \
             合法、可修复       非法
                 ↓               ↓
             Update TT       Terminate
                 ↓               ↓
              Resume           Abort
                 ↓
               Retry
```

具体命令编码、Resume action、事件类型和完成规则，需要以所用 SMMUv3 版本及 MMU-720AE TRM 为准。

## 9. 为什么 Fault Policy 要按 Stream 配置

不同 requester 的能力和实时性要求不同：

```text
GPU
→ 能够等待 Page Fault 修复
→ 适合 Stall Model

简单 DMA Engine
→ 可能不支持长时间挂起
→ 适合 Abort Model
```

所以系统可以概念上配置：

```text
StreamID 10 → Stall
StreamID 20 → Abort
StreamID 30 → Stall
```

即：

> **Fault handling policy 与 Stream/Translation Context 相关，而不是整个 SMMU 只能统一选择一种模式。**

## 10. 与 CPU MMU Fault 的区别

CPU：

```text
MMU Fault
   ↓
当前指令产生同步异常
   ↓
Exception Handler
   ↓
修复
   ↓
Retry Instruction
```

SMMU：

```text
Device Transaction Fault
   ↓
Requester 不能直接进入 CPU Exception Handler
   ↓
SMMU Stall/保存 Transaction
   ↓
通过 Event / Interrupt 通知软件
   ↓
Software 修复或判定非法
   ↓
Resume / Terminate
```

所以：

> **SMMU Fault 本质上是设备 DMA Fault。SMMU 不能让 Device 直接进入 CPU 异常处理程序，因此需要通过 Stall、Event 和 Resume/Terminate 机制，把 Fault Handling 代理给 CPU/software。**

## 面试式回答

> **SMMU 的 Fault Handling 可以采用 Abort 或 Stall。Abort 模式下，translation fault 后直接终止 transaction 并向 upstream master 返回错误；Stall 模式下，SMMU 暂停 faulting transaction，并通过 event/interrupt 通知软件。软件可以建立或修复 translation table，完成必要的失效和同步后让 SMMU Resume/retry；如果访问非法，则选择 Terminate/Abort。Stall 模式因此能够为 Device DMA 提供类似 CPU Page Fault 的 Demand Paging 能力，而且 Fault Model 可以针对不同 Stream 独立配置。**

## 一句话总结

> **Abort = “翻译失败就结束”；Stall = “先挂住，软件修好再继续，非法则终止”。**

## 相关记录与资料

- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [Command、Event 与 PRI Queue](05_Command_Event与PRI_Queue.md)
- [ARM MMU 常见 Fault](13_ARM_MMU常见Fault及产生原因.md)
- [ARM MMU Fault 与定位](../../02_KNOWLEDGE/Debug/01_ARM_MMU_Fault与定位.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

