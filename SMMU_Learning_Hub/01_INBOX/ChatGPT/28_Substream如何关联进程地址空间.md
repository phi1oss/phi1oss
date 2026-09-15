# Substream 如何关联进程地址空间

> 记录日期：2026-09-01  
> 记录端：本地  
> 来源：ChatGPT 对话 `SMMU`  
> 原始上下文：基于此前三页培训视频截图继续追问，截图时间点未知  
> 状态：Inbox 问答整理，待结合 SMMUv3 Substream、Context Descriptor 与 PCIe PASID 规范校验蒸馏

## 凝练问题

**SMMUv3 的 Substream 具体怎么使用？SubstreamID/SSID 如何与一个进程建立联系，使同一个设备能够访问不同进程的 Stage 1 地址空间？**

## 核心结论

> **Substream 是同一个 Device/Stream 内进一步区分“这笔内存访问属于哪个进程地址空间”的第二级身份机制。它与进程的联系不是由 SMMU 自动推断，而是由系统软件预先建立 `SSID → Context Descriptor → ASID/TTBR → Process Page Table` 的绑定，支持 Substream 的设备在发出 Transaction 时再携带对应 SSID。**

最简链路是：

```text
Process
   ↓ 软件建立绑定
SSID
   ↓ 选择
Context Descriptor
   ↓ 保存
ASID + TTBR + Stage 1 Configuration
   ↓
该 Process 的 Stage 1 Page Table
```

一句话记忆：

> **SID 找设备，SSID 找进程地址空间，CD 找到该地址空间的 Stage 1 页表。**

## 1. 为什么有了 StreamID 还需要 SubstreamID

假设一个 GPU 同时为两个进程执行任务：

```text
GPU
├─ Process A
└─ Process B
```

它们都来自同一个物理设备，因此在 SMMU 侧可能具有相同的 StreamID：

```text
SID = 20 → GPU
```

但两个进程可以在相同 VA 上建立不同映射：

```text
Process A：VA 0x4000 → Mapping A
Process B：VA 0x4000 → Mapping B
```

只靠 SID，SMMU 只能识别“请求来自 GPU”，不能区分“GPU 正在代表哪个进程访问”。因此需要：

```text
SID  → 标识 Device/Stream
SSID → 标识该 Stream 内选择的 Process Address Space
```

这里的“标识进程”是帮助理解的简写；更精确地说，SSID 选择的是 **Stage 1 Translation Context**，该 Context 由软件与某个进程地址空间建立绑定。

## 2. SSID 与进程建立联系的本质

以两个进程为例，系统软件为其准备独立的 Stage 1 Context：

```text
Process A
├─ Page Table A
├─ ASID = 10
└─ 分配/绑定 SSID = 3

Process B
├─ Page Table B
├─ ASID = 20
└─ 分配/绑定 SSID = 7
```

软件再建立 Context Descriptor Table：

```text
SSID = 3
   ↓
CD[3]
├─ ASID = 10
├─ TTB0 → PageTable_A
└─ Stage 1 Configuration A

SSID = 7
   ↓
CD[7]
├─ ASID = 20
├─ TTB0 → PageTable_B
└─ Stage 1 Configuration B
```

所以关系不是：

```text
SSID = OS Process ID
```

而是：

```text
SSID
 ↓ 作为索引选择
CD
 ↓ 保存
ASID + TTB + Stage 1 Configuration
 ↓ 指向
Process Page Table
```

这才是 SSID 与进程地址空间建立联系的本质。

具体由 OS、Hypervisor、IOMMU/SMMU Driver 还是 Device Driver 分配和协同编程，取决于系统的软件栈与设备接口；不能把示例中的“OS 分配 SSID”当作所有平台唯一的软件职责划分。

## 3. 谁把 SSID 放进 Transaction

SSID 不是 SMMU 收到请求后生成的，而是由支持 Substream 的上游设备或接口随 Transaction 提供。

例如 GPU 调度到 Process B 的 Workload：

```text
GPU Scheduler
      ↓
选择 Process B 的执行上下文
      ↓
获得该上下文绑定的 SSID = 7
      ↓
设备发出 Transaction：
SID  = 20
SSID = 7
VA   = 0x4000
```

因此要实现正确的进程级地址翻译，需要软件配置与设备发包两侧一致：

```text
软件侧：SSID 7 → CD[7] → PageTable_B
设备侧：Process B 的请求携带 SSID 7
```

如果设备携带了错误的 SSID，SMMU 会选择错误的 Stage 1 Context，或因无效/未配置的 Context 产生 Fault；具体结果取决于相应 STE/CD 配置与架构规则。

## 4. SMMU 收到 SID、SSID 和地址后的查找流程

假设：

```text
SID  = 20
SSID = 7
VA   = 0x4000
```

Stage 1 查找链路为：

```text
Transaction
SID=20, SSID=7, VA=0x4000
          ↓
SID 选择 Stream Table 中的 STE[20]
          ↓
读取 STE.S1ContextPtr
          ↓
定位 Context Descriptor Table
          ↓
SSID=7 选择 CD[7]
          ↓
CD[7] 提供 ASID、TTB、Stage 1 Configuration
          ↓
使用 Process B 的 Stage 1 Page Table
          ↓
VA 0x4000 → IPA（启用 Stage 2 时）
```

如果还启用 Stage 2：

```text
IPA
 ↓
STE[20] 中的 VMID + S2TTB
 ↓
Stage 2 Translation Table
 ↓
PA
```

完整路径可以压缩为：

```text
SID → STE
       ├─ Stage 2：VMID + S2TTB
       └─ Stage 1：S1ContextPtr
                        ↓
                     CD Table
                        ↑
                       SSID
                        ↓
                       CD
                        ↓
                 ASID + TTB + S1 Config
                        ↓
                 Stage 1 Page Table
```

## 5. 为什么多个 Substream 可以共享 Stage 2

考虑同一 VM 内的多个进程：

```text
VM1
├─ Process A
├─ Process B
└─ Process C
```

它们的用户虚拟地址空间不同，因此需要不同的 Stage 1 Context：

```text
SSID A → CD A → Page Table A
SSID B → CD B → Page Table B
SSID C → CD C → Page Table C
```

但如果这些 Substream 都属于同一个 STE 和同一个 VM Translation Context，它们可以共享 STE 中的 Stage 2 配置：

```text
                    SID = 20
                       ↓
                      STE
                ┌──────┴──────┐
                │             │
       Shared Stage 2     S1ContextPtr
        VMID + S2TTB           ↓
                            CD Table
                           /   |   \
                         CD A CD B CD C
                          ↓    ↓    ↓
                         PT A PT B PT C
```

因此可以记为：

> **SSID 主要用于在一个 Stream 内选择不同的 Stage 1 地址空间。**

“共享 Stage 2”是同一 STE/VM Context 下的典型组织方式，不表示架构要求所有 Substream 在所有部署中都必然属于同一 VM 或共享相同 Stage 2。

## 6. PCIe PASID 与 SSID

PCIe PASID（Process Address Space ID）是 Substream/SSID 的典型来源之一，用于标识一笔 PCIe Request 属于哪个 Process Address Space。

概念对应关系是：

```text
PCIe Requester/Device Identity → StreamID
PCIe PASID                     → SubstreamID / SSID
```

于是：

```text
PCIe Accelerator
├─ Process A → PASID 5
└─ Process B → PASID 9

Requester Identity + PASID + Address
                  ↓
               SMMU
                  ↓
SID 选择 STE，SSID/PASID 选择 CD
```

PASID 到 Arm SSID 的具体传递和映射还依赖 PCIe Root Complex、SMMU Integration 与相关协议配置，不能把“概念对应”简单理解为所有系统中位宽和数值都必然一一原样相等。

## 7. SSID 与 ASID 的区别

“SSID mapped to ASID”不等于：

```text
SSID == ASID
```

更准确的关系是：

```text
SSID
 ↓ 选择
CD
 ↓ CD 中包含
ASID
```

二者职责不同：

| 标识 | 主要作用 |
|---|---|
| SID | 选择 Stream/STE |
| SSID | 在该 Stream 内选择 Stage 1 Context/CD |
| ASID | 标记该 Stage 1 Translation Context，参与 TLB 区分和维护 |
| VMID | 标记 Stage 2/VM Translation Context |

所以：

> **SSID 是“选择哪个 CD”的输入标识；ASID 是“这个 CD 所描述的 Stage 1 Translation 在缓存中属于哪个地址空间”的标签。**

## 8. 完整示例

初始化/绑定阶段：

```text
GPU：SID = 20

Process A：
SSID = 3
ASID = 10
TTB  = PageTable_A

Process B：
SSID = 7
ASID = 20
TTB  = PageTable_B

STE[20].S1ContextPtr → CD Table
CD[3] → ASID 10 + PageTable_A
CD[7] → ASID 20 + PageTable_B
```

Process B 运行时：

```text
GPU 发出：
SID=20, SSID=7, VA=0x4000
       ↓
SMMU：STE[20] → CD[7] → PageTable_B
       ↓
VA 0x4000 → IPA_B → Stage 2 → PA_B
```

Process A 即使访问相同 VA：

```text
SID=20, SSID=3, VA=0x4000
       ↓
SMMU：STE[20] → CD[3] → PageTable_A
       ↓
得到不同的 Translation Result
```

## 面试式回答

> **Substream 是 SMMUv3 在同一个 Stream 内进一步区分多个进程地址空间的机制。StreamID 标识 Device/Stream，而设备在 Transaction 中进一步携带 SubstreamID/SSID，表示这笔访问应使用哪个 Process Address Space。系统软件预先建立 SSID 到 Context Descriptor 的映射，每个 CD 保存相应 Stage 1 Context 的 ASID、TTB 和 Translation Configuration。因此 SMMU 收到 SID+SSID 后，先由 SID 找 STE，再由 SSID 从 CD Table 中选择对应 Stage 1 Context。多个 Substream 可以拥有不同的 Stage 1 Page Table，并可在同一 STE/VM Context 下共享 Stage 2。PCIe PASID 是典型的进程级 Substream 标识来源。**

## 一句话总结

> **Substream 与进程的联系不是“SSID 等于 PID”，而是软件建立 `SSID → CD → ASID/TTBR → Process Page Table` 的绑定，设备再用同一 SSID 标记该进程的 Transaction。**

## 相关记录与资料

- [Substream、SSID 与 PCIe PASID](19_Substream_SSID与PASID.md)
- [多进程如何使用 TTBR0_EL1](25_多进程如何使用TTBR0_EL1.md)
- [StreamID 与 Translation Context 映射](18_StreamID与Translation_Context映射.md)
- [Linear 与 2-Level Stream Table](27_Linear与2-Level_Stream_Table.md)
- [SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)
- [MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)

