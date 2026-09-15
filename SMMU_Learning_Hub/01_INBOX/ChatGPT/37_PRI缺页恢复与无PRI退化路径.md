# PRI 缺页恢复与无 PRI 时的退化路径

记录日期：2026-09-03  
记录端：手机 / GitHub；本地整理于 2026-09-03  
来源：[GitHub 手机端记录：PRI Queue 与无 PRI 时 ATS 缺页处理](https://github.com/phi1oss/phi1oss/blob/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93/00_Inbox/2026-09-03_1007_PRI_Queue%E4%B8%8E%E6%97%A0PRI%E6%97%B6ATS%E7%BC%BA%E9%A1%B5%E5%A4%84%E7%90%86.md)  
原始资料：MMU.pdf 物理第 51 页 PRI Queue 培训内容  
状态：Inbox；PCIe PRI record、response code、DTI-ATS message 与 MMU-720AE PRIQ overflow 细节待规范校验

## 问题

支持 ATS 的 Device 查询一个尚未驻留或尚未建立映射的页面时，PRI Queue 如何把缺页请求交给软件？软件为什么既要推进 `PRIQ_CONS`，又要发送 `CMD_PRI_RESP`？如果系统不支持 PRI，又该如何处理 ATS 缺页？

## 回答

### 核心结论

> **ATS 负责“问地址怎么翻”，PRI 负责“现在翻不了，请软件把页面准备好”。没有 PRI 时，ATS 仍能报告 translation failure，但不存在标准的 Device→PRI→Software demand-paging 恢复闭环。**

## 1. PRI Queue 的角色

PRI（Page Request Interface）是 ATS/SVA 体系中的可选缺页请求机制。PCIe Device 通过 Root Complex 和 DTI-ATS 发送 Page Request，SMMU 将请求写入内存中的 PRI Queue（PRIQ），再由软件处理。

```text
PRIQ Producer = SMMU
PRIQ Consumer = Software
```

- `SMMU_PRIQ_PROD`：SMMU 写入新的 PRI Entry 后推进。
- `SMMU_PRIQ_CONS`：软件消费对应 Entry 后推进。

PRIQ 保存的是 Page Request record/metadata，不保存原始 DMA payload。

## 2. ATS 缺页到 PRI 恢复的完整闭环

```text
PCIe Device
    │ ATS Translation Request
    ▼
Root Complex ──DTI-ATS──> SMMU
                           │
                           │ Page not present / 无法完成 translation
                           ▼
Device 收到 ATS failure
    │
    │ PRI Page Request
    ▼
Root Complex ──DTI-ATS──> SMMU
                           │ 写入 PRI Entry，推进 PRIQ_PROD
                           ▼
                        PRI Queue
                           │ IRQ
                           ▼
Software
    │ 检查请求是否合法
    │ 分配或调入 page
    │ 建立/更新页表映射
    │ 完成必要的 translation maintenance
    │ 消费 Entry，推进 PRIQ_CONS
    │ 通过 CMDQ 提交 CMD_PRI_RESP
    ▼
SMMU → Root Complex → PCIe Device
    │
    ├─ Success：Device 重新请求 translation / 继续访问
    └─ Failure：结束该 Page Request
```

准确的 request grouping、failure encoding、重试要求与 response code 应以 PCIe PRI/SMMUv3 规范为准；上图用于理解控制流。

## 3. PRIQ_CONS 与 CMD_PRI_RESP 为什么不能互相替代

推进 `PRIQ_CONS` 只表示：

> **软件已经读取并处理完这个内存队列条目，队列空间可以回收。**

它不会自动把处理结果送回 PCIe Endpoint。`CMD_PRI_RESP` 才表示：

> **软件明确返回这次 Page Request 的 Success、Failure 等协议结果。**

因此：

```text
PRIQ / PRIQ_CONS
→ SMMU 与 Software 之间的队列消费关系

CMD_PRI_RESP
→ Software 经 SMMU 向 Device 返回 Page Request 结果
```

## 4. 没有 PRI 时会发生什么

没有 PRI 时，ATS translation 遇到 page not present 仍可以失败：

```text
ATS Translation Request
        ↓
SMMU Translation
        ↓
Page not present
        ↓
ATS Completion：Failure
        ↓
没有标准 PRI demand-paging 恢复通道
```

系统不能期待 SMMU 自动创建 PRI Request、唤醒 page-fault handler 或向 Device 返回 `CMD_PRI_RESP`。

### 路径一：提前准备并保持页面可用

```text
Driver / OS
   ↓ 提前分配或 fault-in page
建立 SMMU mapping
   ↓ 必要时 pin / 保持 resident
完成所需 TLBI / SYNC / ATC maintenance
   ↓
启动 Device workload
```

这种模式把运行时 demand paging 改为启动前准备，确保 Device 查询时 mapping 已经存在。

### 路径二：改用普通 Untranslated DMA 的 Stall Fault 恢复

如果 Stream 支持 Stall Model，普通 Untranslated transaction 可以走：

```text
Untranslated DMA
      ↓
SMMU translation fault 并暂挂 transaction
      ↓
EVTQ 报告 Software
      ↓
Software 修复页表
      ↓
CMD_RESUME
```

这与 PRI 都能实现“fault 后由软件修复”，但协议对象不同：

- **PRI**：Device 主动发 Page Request，面向 ATS/SVA 的设备侧缺页恢复。
- **Stall Model**：SMMU 暂挂普通 transaction，并通过 EVTQ 报告软件。

> **Stall 是另一条 fault-recovery 路径，不是 PRI 的协议替代物。**

## 5. 与 SVA 的关系

SVA 允许 Device 使用进程 VA，而用户页可能暂时不在内存。ATS + PRI 可以形成 Device-side demand paging：

```text
Device 使用 Process VA
      ↓ ATS：page not present
PRI Page Request
      ↓
OS Page Fault Handler 准备页面
      ↓
更新 Process Page Table
      ↓
CMD_PRI_RESP
      ↓
Device 重试 ATS
```

如果没有 PRI，SVA 的设备侧 demand paging 能力会受限，通常需要预先 fault-in/pin 相关用户页，或选择其他访问与恢复模型。

## 6. 集成与验证关注点

- SMMU/MMU-720AE 是否实现并启用 PRI/PRIQ。
- Endpoint、Root Complex 和 DTI-ATS 是否完整支持 Page Request/Response。
- PRIQ Base、PROD、CONS、IRQ 与 overflow 行为是否配置正确。
- 软件是否在准备 mapping 后完成所需维护，再提交 `CMD_PRI_RESP`。
- PRI unsupported 时，不应错误地产生 PRIQ Entry；软件必须有页面驻留策略或另一条 fault-recovery 路径。
- Page Request 失败时，Device 不得继续把对应地址当作有效 translation 使用。

## 面试版总结

> **PRI 是 PCIe ATS 的缺页恢复机制。ATS translation 因 page not present 失败后，支持 PRI 的 Device 可以发送 Page Request；SMMU 将其写入 PRIQ，软件准备页面并更新映射，然后通过 CMDQ 提交 CMD_PRI_RESP，把成功或失败返回 Device。推进 PRIQ_CONS 只代表软件消费了队列条目，不等于 Response 已送达 Device。若系统不支持 PRI，就无法使用标准的设备侧 demand paging，通常需要提前建立并保持 mapping，或让普通 DMA 走 SMMU Stall/EVTQ/Resume 路径。**

## 一句话总结

> **ATS 是“问地址怎么翻”，PRI 是“翻不了，请 OS 补页”；没有 PRI，就要提前把页面准备好，或使用另一条 fault-recovery 路径。**

## 关联记录

- [Command Queue、Event Queue 与 PRI Queue](05_Command_Event与PRI_Queue.md)
- [SMMU Fault Model：Abort、Stall 与 Resume](21_SMMU_Fault_Model_Abort_Stall与Resume.md)
- [EVTQ 与 Global Error 的职责边界](34_EVTQ与Global_Error的职责边界.md)
- [PCIe ATS 与 ATC 工作流程](35_PCIe_ATS与ATC工作流程.md)
- [SVA：共享虚拟地址与进程—设备绑定](38_SVA共享虚拟地址与进程设备绑定.md)

