# SVA：共享虚拟地址与进程—设备绑定

记录日期：2026-09-03  
记录端：手机 / GitHub；本地整理于 2026-09-03  
来源：[GitHub 手机端记录：SVA 共享虚拟地址与进程—设备绑定](https://github.com/phi1oss/phi1oss/blob/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93/00_Inbox/2026-09-03_1110_SVA%E5%85%B1%E4%BA%AB%E8%99%9A%E6%8B%9F%E5%9C%B0%E5%9D%80%E4%B8%8E%E8%BF%9B%E7%A8%8B%E8%AE%BE%E5%A4%87%E7%BB%91%E5%AE%9A.md)  
原始资料：MMU.pdf 物理第 55–57 页 SVA 培训内容  
状态：Inbox；PASID/SSID 传递、SVA Bind 软件接口、页表共享条件和设备生命周期待结合 SMMUv3、PCIe 与操作系统实现校验

## 问题

SVA 到底共享了什么？同一个 Device 同时服务多个进程时，SMMU 如何知道某个 VA 属于哪个进程？SVA Bind、SID/SSID、STE/CD、ASID/TTBR、ATS 和 PRI 分别承担什么职责？

## 回答

### 核心结论

> **SVA 不是一种新的地址翻译算法，而是把 Device 绑定到某个 Process Address Space，使 Device 代表该进程访问内存时可以直接使用进程 VA，并受该进程页表 mapping 和 permission 的约束。**

## 1. 传统 DMA 与 SVA 的区别

传统 DMA 中，CPU 与 Device 往往使用不同的地址语义：

```text
Application VA ──CPU Page Table──> PA
Device IOVA    ──SMMU Page Table─> PA
```

应用指针通常需要经过 Driver/DMA API 转换成 Device 使用的 DMA address。

SVA 希望建立：

```text
Application pointer / Process VA
         ├─ CPU 使用
         └─ Device 代表该 Process 使用
```

其价值是让 CPU 和 Device 共享同一进程虚拟地址空间的语义，减少额外的 Device IOVA 管理与指针转换，并方便 GPU、AI accelerator、NIC 等设备直接处理用户态复杂数据结构。

## 2. 为什么仅有 VA 不够

不同进程可以拥有相同 VA，但映射到不同物理页面：

```text
Process A：VA 0x4000 → PA_A
Process B：VA 0x4000 → PA_B
```

因此 Device transaction 还必须携带它当前代表的地址空间身份：

```text
StreamID       → 哪个 Device / Stream
PASID / SSID   → 该 Device 当前代表哪个 Process Address Space
Address        → 该 Process 的 VA
```

一句话可以记为：

> **SID 找设备，SSID 找进程地址空间。**

PCIe 侧通常使用 PASID；进入 SMMUv3 后，对应 SubstreamID/SSID。具体映射、位宽和数值传递取决于 Root Complex 与 SMMU 集成，不能假定所有系统都简单原样透传。

## 3. SID、SSID、STE 与 CD 的查找链

```text
Device transaction：SID + SSID/PASID + VA
        │
        ├─ SID 选择 STE
        │      ├─ Stream 级行为
        │      ├─ Stage 2：VMID + S2TTB（若启用）
        │      └─ Stage 1 Context Descriptor Table 入口
        │
        └─ SSID 选择 CD
               ├─ ASID
               ├─ TTBR / Page Table Base
               └─ Stage 1 translation controls
                        ↓
                  Process Page Table
```

所以一个 Device 可以用同一个 SID、不同 SSID 同时代表多个进程：

```text
SID 20 + SSID 10 → Process A 的 CD / Page Table
SID 20 + SSID 11 → Process B 的 CD / Page Table
```

SSID 不是操作系统 PID，也不等于 ASID：

```text
SSID ──选择──> CD ──包含──> ASID + TTBR
```

## 4. SVA Bind 的本质

SVA Bind 是系统软件建立 `Device + Process Address Space` 绑定的过程。概念上：

```text
Process A
├─ Process page table / mm context
├─ ASID
└─ 分配的 PASID/SSID
        │
        │ SVA Bind
        ▼
STE[Device SID]
        ↓
CD[Process SSID]
├─ ASID_A
├─ TTBR → Process A Page Table
└─ Stage 1 controls
```

绑定后，Device 使用 `SID + SSID + VA` 发起访问，SMMU 才能选择正确的进程 Stage 1 context。

“SVA 让进程控制 Device”这一说法不够准确。更准确的是：

> **SVA 让 Device 在代表某个 Process 访问内存时复用该 Process 的地址空间，并接受对应页表权限检查。**

例如 Process 页表把一个 VA 配成只读，则 Device 代表该 Process 写该 VA 时，也应受到相应 permission 限制。

## 5. SVA、ATS 与 PRI 的分工

三者解决不同问题：

```text
SVA
→ Device 使用哪个 Process Address Space

ATS
→ Device 如何提前查询并缓存该 VA 的 translation

PRI
→ 该 VA 对应页面尚未准备好时，如何请求 OS 补页
```

组合流程可以是：

```text
Device：VA + PASID
      ↓ ATS Translation Request
Root Complex / SMMU
      ↓ SID → STE；SSID → CD；CD → Process Page Table
Translation Result
      ↓
Device ATC
```

若 PTE not present 且整条链路支持 PRI：

```text
ATS failure
      ↓
PRI Page Request → SMMU PRIQ
      ↓
OS 准备页面并更新 Process Page Table
      ↓
CMD_PRI_RESP
      ↓
Device 重试 ATS
```

所以 SVA 可以独立表达“共享地址空间”的目标，但要实现高性能 translation caching 和 Device-side demand paging，通常还需要 ATS 与 PRI 配合。

## 6. SVA 与 Stage 2 并不冲突

虚拟化环境可以同时保留两个隔离层次：

```text
Process VA
   ↓ Stage 1：SSID → CD → ASID/TTBR
IPA
   ↓ Stage 2：STE → VMID/S2TTB
PA
```

- Stage 1 描述 Process Address Space；不同 SSID 可选择不同 CD。
- Stage 2 描述 VM/Host 隔离；同一 STE 下的多个 Substream 通常可共享该 Stage 2 context。

因此 SVA 共享进程虚拟地址语义，并不意味着放弃 Hypervisor 对最终 PA 的控制。

## 7. 安全性与生命周期

SVA 不表示 Device 可以访问任意进程。系统软件仍需控制：

- PASID/SSID 的分配、授权和回收。
- SVA Bind/Unbind 与 CD 的创建和失效。
- Process page table 与 address-space object 的生命周期。
- Page-table update 后的 CPU TLB、SMMU TLB 和 Device ATC 协同失效。
- Process exit、地址空间释放、Device reset 和重新分配时的 outstanding access。
- 虚拟化场景中的 Stage 2 isolation。

只有软件为 Device 建立的合法 Process Context 才能被使用；确切的软件职责划分取决于 OS、IOMMU framework、Device Driver 和硬件接口。

## 8. 完整模型

```text
Process / Application
├─ ASID
├─ PASID
└─ Page Tables
       ▲
       │ SVA Bind
       │
Device ▼                    SMMU
├─ StreamID ───────────────> STE ── Stage 2 context（可选）
├─ PASID/SSID ─────────────> CD
└─ Process VA                ├─ ASID
                             └─ TTBR ──> Process Page Table

ATS：把 translation result 缓存到 Device ATC
PRI：Device 缺页时请求 OS 准备 page
```

## 面试版总结

> **SVA 让 Device 直接使用某个 Process 的虚拟地址空间。StreamID 标识 Device/Stream，PASID/SSID 标识 Device 当前代表的 Process Address Space；SMMU先用 SID 找 STE，再用 SSID 找 CD，CD 中保存 ASID、TTBR 等 Stage 1 context，从而用相应 Process Page Table 对 Device VA 做 translation 和 permission check。SVA 常与 ATS、PRI 配合：ATS 加速 translation，PRI 支持 Device-side demand paging；在虚拟化系统中仍可保留 Stage 2 作为 VM/Host 隔离边界。**

## 一句话总结

> **SVA 是 Device 直接使用 Process VA；SID 找设备，SSID 找进程，CD 找页表，ATS 加速翻译，PRI 处理缺页。**

## 关联记录

- [Substream 如何关联进程地址空间](28_Substream如何关联进程地址空间.md)
- [Substream、SSID 与 PCIe PASID](19_Substream_SSID与PASID.md)
- [PCIe ATS 与 ATC 工作流程](35_PCIe_ATS与ATC工作流程.md)
- [PRI 缺页恢复与无 PRI 时的退化路径](37_PRI缺页恢复与无PRI退化路径.md)
- [Full ATS 的安全边界](36_Full_ATS的安全边界.md)

