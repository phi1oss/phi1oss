# SMMUv3 寄存器空间与 Memory-based 配置

> 记录日期：2026-09-01  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 原始上下文：培训视频截图，时间点未知  
> 状态：Inbox 问答整理，待结合 SMMUv3 Architecture Specification 与 MMU-720AE TRM 校验蒸馏

## 凝练问题

**如何理解培训截图中 SMMUv3 寄存器空间的组织方式？这些寄存器在整个 Programming Model 中分别承担什么职责，它们与内存中的 Stream Table、STE、CD、Translation Table、CMDQ 和 EVTQ 是什么关系？**

## 核心结论

> **SMMUv3 用寄存器完成 Capability Discovery、全局控制、Queue/Stream Table 的入口配置以及 Error/Interrupt 管理；大量 Per-stream Translation Context 并不直接存放在寄存器中，而是保存在内存中的 STE、CD 和 Translation Table 等结构里。Secure 与 Non-secure 还拥有相互隔离的寄存器视图。**

一句话理解其设计：

> **Registers 管“入口和总控”；STE/CD 管“怎么翻”；Page Table 管“翻到哪”；CMDQ/EVTQ 管“软件与 SMMU 如何通信”。**

## 1. 两层配置模型

这页的重点不是死记寄存器 Offset，而是理解两层配置模型：

```text
               CPU Software
                    │
             MMIO 访问 SMMU
                    ↓
              SMMU Registers
                    │
      ┌─────────────┼─────────────┐
      │             │             │
 Stream Table    Command Q     Event Q
 base/size       base/size     base/size
      │
      ↓
System Memory
 ├─ Stream Table / STE
 ├─ Context Descriptor / CD
 ├─ Translation Tables
 ├─ Command Queue
 └─ Event Queue
```

因此：

- 寄存器主要保存全局控制、能力信息及内存结构的入口。
- 大规模的 Per-stream/Per-context 配置主要保存在 Memory-based Structures 中。

这种设计避免为每个 Translation Context 准备大量固定寄存器，使 SMMUv3 更容易扩展到大量 Stream 和 Context。

## 2. Secure 与 Non-secure 寄存器视图

培训材料区分：

```text
SMMU_S_* → Secure Register View
SMMU_*   → Non-secure Register View
```

Secure 寄存器对不具备相应权限的 Non-secure Software 不可用于修改 Secure SMMU Configuration。逻辑关系是：

```text
Secure Software                 Non-secure Software
       ↓                               ↓
SMMU_S_* Register View          SMMU_* Register View
       ↓                               ↓
Secure Configuration            Non-secure Configuration
```

这与 Secure Stream Table 和 Non-secure Stream Table 的隔离思想一致：Non-secure Software 不能通过自己的配置路径破坏 Secure Translation Environment。

## 3. 主要寄存器类别

### ID Registers

用于 Capability Discovery，回答“这个 SMMU 实现支持什么”，例如：

```text
Stage 1 / Stage 2
ATS / PRI / HTTU
ASID16 / VMID16
其他 Optional Feature
```

ID Register 描述能力，不等同于 Enable 开关。

### Control Registers

负责 SMMU 的全局运行控制，例如概念上的：

```text
SMMU Enable
CMDQ Enable
EVTQ Enable
其他 Global Control
```

可以记为：

```text
ID Registers      → “你会什么？”
Control Registers → “现在让你做什么？”
```

### Interrupt Registers

负责配置 SMMU 如何通过中断向 CPU 报告 Event/Error：

```text
SMMU Event / Error
       ↓
Interrupt Configuration
       ↓
GIC
       ↓
CPU Software
```

### Global Error Registers

用于报告 SMMU 自身的全局运行、编程或 Queue 类错误。需要与 EVTQ 区分：

```text
EVTQ
→ 主要承载架构定义的 Transaction/Event Record

Global Error Registers
→ 反映 SMMU 全局运行或编程状态错误
```

## 4. Stream Table Registers

Stream Table Registers 不保存具体 STE，而是回答：

> **Stream Table 在内存的什么位置、大小是多少、采用什么组织格式？**

例如：

```text
SMMU_STRTAB_BASE
→ Stream Table Base Address

SMMU_STRTAB_BASE_CFG
→ Stream Table Size / Format
```

运行时：

```text
Incoming Transaction 携带 StreamID
               ↓
根据 STRTAB_BASE 找到 Stream Table
               ↓
StreamID 选择对应 STE
```

关键边界是：

> **Register 指向 Stream Table；STE 本身位于 Memory。**

## 5. Command Queue 与 Event Queue Registers

### Command Queue Registers

它们配置内存 CMDQ 的 Base、Size 和 Queue 状态/指针，而不是把全部 Command 存放在寄存器中：

```text
SMMU_CMDQ_BASE
      ↓
Memory-based CMDQ
 ├─ TLBI
 ├─ CFGI
 ├─ SYNC
 └─ 其他 Management Command
```

数据方向：

```text
Software 写 CMDQ
       ↓
SMMU 取 Command 并执行
```

### Event Queue Registers

EVTQ 的方向与 CMDQ 相反：

```text
CMDQ：Software → SMMU
EVTQ：SMMU → Software
```

例如：

```text
Device DMA
   ↓
SMMU Translation Fault
   ↓
生成 Event Record
   ↓
写入 Memory-based EVTQ
   ↓
通过中断通知 CPU Software
```

EVTQ Registers 用来配置 Queue 的位置、大小及 Producer/Consumer 状态。

## 6. Address Translation 与 Implementation-defined Registers

### Address Translation Registers

用于架构定义的软件 Address Translation/Translation Query 类操作，但其中一些能力可能是 Optional，必须先检查 ID Registers。

例如此前学习 MMU-720AE 时看到：

```text
SMMU_IDR0.ATOS = 0
```

这说明不能因为架构寄存器空间中预留了相应区域，就认为所有 SMMUv3 实现都支持全部软件地址翻译操作。

### Implementation-defined Registers

具体 IP 可以在架构允许的区域中加入实现相关寄存器，例如：

```text
Debug
FuSa
Performance
Integration Control
Implementation-specific Error
```

这类内容不能只查通用 SMMUv3 Architecture Specification，还需要查看具体实现的 TRM，例如 MMU-720AE TRM。

## 7. 与 STE、CD 和 Page Table 串联

```text
                  CPU Software
                      │
                      ↓
                SMMU Registers
                      │
          配置 Base / Size / Enable
                      ↓
                   Memory
                      │
       ┌──────────────┼──────────────┐
       │              │              │
 Stream Table       CMDQ           EVTQ
       │
       ↓
      STE
       │
   ┌───┴────┐
   │        │
Stage 2   S1ContextPtr
              ↓
             CD
              ↓
      Stage 1 Translation Table
```

可以形成四层理解：

```text
Registers → SMMU 去哪里找配置、是否启用？
STE       → 这个 Stream 采用什么 Translation Mode？
CD        → 这个 Substream 的 Stage 1 如何翻译？
Page Table→ 具体地址映射到哪里？
```

## 面试式回答

> **SMMUv3 的寄存器主要负责 Capability Discovery、全局控制、Interrupt/Error 管理，以及配置 Stream Table、Command Queue、Event Queue 等 Memory-based Structures 的地址和大小。Secure 和 Non-secure 拥有独立的寄存器视图，避免 Non-secure Software 修改 Secure Configuration。具体 Per-stream Translation Context 并不主要保存在寄存器中，而是由 StreamID 选择内存中的 STE，Stage 1 再通过 CD 指向相应 Translation Table。**

## 一句话总结

> **SMMUv3 Registers 保存“入口和总控”，大规模 Translation Configuration 与 Queue 内容保存在 Memory；SID 找 STE，STE/CD 决定怎么翻，Page Table 决定翻到哪里。**

## 相关记录与资料

- [SMMUv3 Programming Model](23_SMMUv3_Programming_Model.md)
- [SMMUv3 为什么使用 Command Queue 进行维护](24_SMMUv3为何使用Command_Queue维护.md)
- [StreamID 与 Translation Context 映射](18_StreamID与Translation_Context映射.md)
- [SMMU 对 TrustZone 的支持](20_SMMU对TrustZone的支持.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

