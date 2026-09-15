# SMMUv3 Programming Model

> 记录日期：2026-08-30  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 来源材料：培训截图，时间点未知  
> 状态：Inbox 问答整理，待结合 SMMUv3 规范查证寄存器与队列细节

## 凝练问题

**软件通过哪些接口配置和管理 SMMUv3？Register File、Stream Table、STE、CD、Translation Table、Command Queue 与 Event Queue 分别承担什么职责，它们如何组成完整的编程与运行闭环？**

## 核心结论

> **SMMUv3 的编程模型是“少量寄存器 + 大量 memory-based data structures + command/event queues”。软件通过寄存器告诉 SMMU 各种内存结构的位置，通过 Stream Table/STE/CD 描述每个 Stream 的 Translation Context，通过 Command Queue 下发维护命令，再通过 Event Queue 接收 fault/event。**

## 1. 为什么大部分配置存放在 Memory

SMMUv3 不为每个 Translation Context 准备一整组固定寄存器，而是将大量配置放在系统内存中：

```text
System Memory
├─ Stream Table
│   ├─ STE
│   ├─ STE
│   └─ ...
│
├─ Context Descriptor Table
│   ├─ CD
│   ├─ CD
│   └─ ...
│
├─ Translation Tables
├─ Command Queue
├─ Event Queue
└─ PRI Queue（支持和启用时）
```

软件负责：

```text
Allocate Memory
      ↓
Populate STE / CD / Queues / Translation Tables
      ↓
通过 SMMU Register File
配置各结构的 Base Address、Size 和 Enable
```

这种 memory-based configuration model 使 SMMUv3 能够扩展到大量 Device、Stream、Substream 和虚拟机 Context，而不需要为每个 Context 固定占用大量硬件寄存器。

## 2. Register File 的职责

Register File 是 SMMU 的全局控制入口，概念上负责：

```text
Stream Table Base / Format
Queue Base / Size
SMMU Enable
Queue Producer / Consumer 状态
Interrupt / Queue Control
Capability / ID Registers
Global Bypass / Error Control
```

软件通过 MMIO 访问：

```text
CPU Software
     ↓ MMIO
SMMU Register File
     ↓
告诉 SMMU：
“配置结构在哪里、规模多大、何时启用”
```

关键边界是：

> **寄存器主要保存全局控制和 memory structure 的入口信息，而不是保存每个 Device 的完整 Translation Context。**

## 3. Command Queue：Software → SMMU

Command Queue 是位于内存中的 circular queue：

```text
Software
   ↓ 写入命令
Command Queue
   ↓ SMMU 读取执行
SMMU
```

概念结构：

```text
CMDQ
├─ TLBI ...
├─ CFGI ...
├─ CMD_SYNC ...
├─ PRI Response ...
└─ 其他 SMMU Management Command
```

它主要承担管理和维护，而不是普通 Device Translation 数据通路，例如：

- Translation/TLB invalidation。
- Configuration Cache invalidation。
- 命令同步和完成确认。
- Page Request 响应。
- 其他架构定义的管理操作。

所以：

> **Command Queue 是 Software 控制 SMMU 的异步命令通道。**

具体命令名称、操作数、合法顺序和完成条件，应以目标 SMMUv3 版本和 MMU-720AE TRM 为准。

## 4. Event Queue：SMMU → Software

Event Queue 的方向与 CMDQ 相反：

```text
Device Transaction
      ↓
     SMMU
      ↓ Fault / Event
Event Queue
      ↓ Software 读取
CPU Software
```

SMMU 可以写入的事件包括：

```text
Translation Fault
Permission Fault
Configuration Error
其他 Architectural Event
```

通常还会配合中断通知 CPU：

```text
SMMU
 ├─ 写 Event Queue
 └─ 触发 Interrupt
          ↓
     CPU Software
```

方向记忆：

```text
CMDQ：Software → SMMU
EVTQ：SMMU → Software
```

## 5. Stream Table：决定 Transaction 如何翻译

Incoming transaction 携带 Address 和 StreamID：

```text
Device Transaction
      ├─ Input Address
      └─ StreamID
            ↓
        Stream Table
            ↓
           STE
```

概念上，StreamID 是选择 Stream Table Entry 的身份信息：

```text
SID = 10
   ↓
选择 STE[10]
```

STE 决定该 Stream 的总体 translation mode：

```text
Bypass
Stage 1 Only
Stage 2 Only
Stage 1 + Stage 2
```

## 6. STE 的职责

STE 是 Stream 级的总体 Translation Configuration 入口，主要包括：

```text
STE
├─ Config
│   ├─ Bypass
│   ├─ Stage 1 Only
│   ├─ Stage 2 Only
│   └─ Stage 1 + Stage 2
│
├─ Stage 2 Table Base
├─ Stage 2 VMID
├─ Stage 2 Granule / Address Size / Attributes
├─ Stage 1 CD / CD Table Pointer
└─ 其他 Stream 级控制
```

所以：

> **STE 定义 Stream 的翻译大方向，尤其承载 Stage 2 配置，并在启用 Stage 1 时告诉 SMMU 去哪里找到 CD。**

## 7. Context Descriptor 的职责

CD 描述具体的 Stage 1 Translation Context：

```text
StreamID
   ↓
STE
   ↓
CD / CD Table
   ↓
Stage 1 Translation Table
```

CD 通常包含：

```text
Stage 1 Page-table Base
ASID
Translation Granule
Input Address Size
Table-walk Cacheability / Shareability
其他 Stage 1 Translation Control
```

所以：

> **CD 负责描述 Stage 1 “怎么翻”，而 Stage 1 Translation Table 保存具体的 IOVA→IPA/PA Mapping。**

如果存在 Substream：

```text
StreamID
   ↓
STE
   ↓
CD Table
   ↑
SSID 选择具体 CD
```

## 8. StreamID、STE、CD 与 Page Table 的完整关系

```text
Device Transaction
      ├─ StreamID
      └─ IOVA
          ↓
      Stream Table
          ↓
         STE
      ┌────┴─────┐
      │          │
Stage 2 Config   CD / CD Table Pointer
      │          ↓
      │         CD
      │          ↓
      │   Stage 1 Configuration
      │          ↓
      │   Stage 1 Translation Table
      │
      └───────────────┐
                      ↓
             Address Translation
                      ↓
                     PA
```

这几类结构的职责不能混淆：

```text
STE / CD
→ Translation Configuration：怎么翻

Translation Table
→ Address Mapping：翻到哪里

Register File
→ Global Control 和内存结构入口
```

## 9. Programming 阶段与 Runtime Translation

### Software Programming

```text
CPU Software
    ↓
Allocate Stream Table / CD / Translation Tables / Queues
    ↓
Fill STE / CD / Queue State
    ↓
Write SMMU Registers
    ↓
Enable SMMU
```

这一阶段的目标是：

> **建立 Translation Configuration 和软件—SMMU 通信结构。**

### Device Runtime Translation

```text
Device DMA
   ↓
SID + IOVA
   ↓
SMMU
   ↓
STE / CD
   ↓
TLB Lookup / Translation Table Walk
   ↓
PA + Attributes
```

这一阶段的目标是：

> **使用已经建立的配置完成真实设备 transaction 的地址翻译与保护检查。**

## 10. 修改 Mapping 的软件闭环

假设软件为 GPU 新增：

```text
IOVA 0x4000 → PA 0x80004000
```

概念流程是：

```text
1. 修改 Translation Table
       ↓
2. 必要时修改 STE / CD
       ↓
3. 保证内存写入满足可见性和顺序要求
       ↓
4. 通过 CMDQ 下发适用的 TLBI / CFGI
       ↓
5. 下发 CMD_SYNC 并等待完成
       ↓
6. 新 Configuration / Translation 生效
```

之后 GPU 的新请求：

```text
SID + IOVA
   ↓
SMMU
   ↓
使用新的 Translation
```

如果发生错误：

```text
SMMU
   ↓
Event Queue
   ↓
Interrupt
   ↓
CPU Software 处理
```

最终形成完整控制闭环：

```text
          Software
          /      \
         ↓        ↑
       CMDQ      EVTQ
         ↓        ↑
         └── SMMU ──┘
              ↓
          STE / CD
              ↓
      Translation Tables
              ↓
       Device Translation
```

## 面试式回答

> **SMMUv3 使用 memory-based configuration model。软件先在内存中分配 Stream Table、Context Descriptor、Translation Table、Command Queue 和 Event Queue，再通过 SMMU Register File 配置这些结构的 base address、size 和 enable 状态。运行时，incoming transaction 的 StreamID 用来选择 STE，STE 定义总体 translation mode 和 Stage 2 配置，并在需要 Stage 1 时指向 CD，由 CD 定义 Stage 1 Translation Context。软件通过 Command Queue 下发 TLBI、configuration invalidation 和 synchronization 等维护命令，SMMU 则通过 Event Queue 向软件报告 translation fault 等事件。**

## 一句话总结

> **Register File 决定“配置结构在哪里”；STE/CD 决定“怎么翻”；Translation Table 决定“翻到哪”；CMDQ 表示“软件要求 SMMU 做什么”；EVTQ 表示“SMMU 告诉软件发生了什么”。**

## 相关记录与资料

- [Command、Event 与 PRI Queue](05_Command_Event与PRI_Queue.md)
- [STE 与 CD 的关系](03_STE与CD的关系.md)
- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [StreamID 与 Translation Context 映射](18_StreamID与Translation_Context映射.md)
- [SMMU Fault Model：Abort、Stall 与 Resume](21_SMMU_Fault_Model_Abort_Stall与Resume.md)
- [SMMU Bypass 模式](22_SMMU_Bypass模式.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

