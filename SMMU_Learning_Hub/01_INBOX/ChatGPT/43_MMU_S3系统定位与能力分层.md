# MMU S3 的系统定位与能力分层

记录日期：2026-09-04  
记录端：手机 / GitHub；本地整理于 2026-09-06  
来源：[GitHub 手机端原记录](https://github.com/phi1oss/phi1oss/blob/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93/00_Inbox/2026-09-04_1532_MMU_S3%E7%B3%BB%E7%BB%9F%E5%AE%9A%E4%BD%8D%E4%B8%8ESMMU%E5%9F%BA%E7%A1%80.md)，Training `MMU.pdf` 物理页 59–60  
状态：Inbox；MMU S3 可选能力及 MMU-720AE 的实际实现范围仍需结合产品 TRM 和项目配置核实

## 问题

MMU、SMMU 与 MMU S3 在系统中的定位分别是什么？MMU S3 相比传统 SMMU 增加了哪些与 Arm RME/CCA 相关的能力？

## 回答

### 核心结论

> **CPU MMU 管理处理器发出的地址访问，SMMU 把类似的翻译、权限与隔离能力扩展到 Device/DMA；MMU S3 则进一步把设备侧地址管理纳入 Arm RME/CCA 的安全体系。**

### 1. MMU 不只是做地址替换

MMU 对一次访问通常承担三类职责：

1. **Address Translation**：把输入地址转换为输出地址。
2. **Permission Check**：判断读、写、执行及特权级等访问是否被允许。
3. **Memory Attribute Conversion**：确定输出访问的 memory type、cacheability、shareability 等语义。

因此，MMU 回答的不只是“地址在哪里”，还包括“是否允许访问”和“以什么内存语义访问”。

### 2. CPU MMU 与 SMMU 的职责边界

```text
CPU load/store/fetch → CPU MMU → PA
Device/DMA request   → SMMU    → PA
```

CPU MMU 服务于处理器的指令取值和数据访问。SMMU（System MMU）位于 I/O 请求路径上，为外设、DMA engine、PCIe Device 等 requester 提供地址翻译、权限检查和虚拟化隔离。

二者采用相似的页表和地址空间思想，但请求来源、标识方式以及软件管理接口不同。

### 3. SMMU 支持的地址翻译层级

SMMU 可以按系统配置执行：

- **Stage 1 only**：通常把 Device VA/IOVA 翻译到 IPA 或 PA，描述进程或软件地址空间。
- **Stage 2 only**：把 IPA 翻译到 PA，常用于 VM/Host 隔离。
- **Stage 1 + Stage 2**：先完成进程/Guest 地址空间翻译，再施加 Hypervisor/Host 的物理隔离。

在支持 Substream 的场景中，可以用下列链路理解 Stage 1 context 的选择：

```text
SID → STE → SSID/PASID → CD → ASID + TTBR + translation controls
```

这里是概念链路；PASID 到 SSID 的具体映射由 PCIe Root Complex、互连和 SMMU 的系统集成决定。

### 4. MMU S3 与 RME/CCA 的关系

Training 材料把 MMU S3 描述为面向新一代系统安全模型的 MMU/SMMU 架构。除传统地址翻译外，它还可涉及：

- **GPC/GPT**：检查输出 Physical Address 所在 granule 的 Physical Address Space（PAS）归属。
- **Realm translation 与 Realm Device assignment 相关控制**：支持 Realm/CCA 场景下的设备隔离。
- **DPT**：在相应版本和实现中，为某些设备访问补充 Device/VM 级物理权限检查。
- **MEC**：与内存加密上下文相关的架构能力。

这些名称描述的是架构或 Training 所覆盖的能力集合，不能据此推断当前 MMU-720AE 版本或当前项目已经启用了全部能力。

### 5. 把 MMU S3 放回完整访问流程

```text
Device transaction
        ↓
识别 SID / SSID
        ↓
选择 STE / CD / translation context
        ↓
Stage 1 / Stage 2 translation + permission checks
        ↓
得到 PA
        ↓
GPC/GPT 检查 PA granule 的 PAS 归属（启用时）
        ↓
形成最终 memory access 或报告 fault
```

Translation Table 回答“地址翻译到哪里”，GPC/GPT 回答“这块物理内存在安全意义上属于哪个 PAS、当前请求是否可以访问”。两者是不同层次的检查。

### 证据边界

- **Training 明确内容**：MMU 的基本职责、SMMU 的设备侧定位、MMU S3 与 RME/CCA 能力的系统级关系。
- **架构背景**：Stage 1/Stage 2、PAS、GPT/GPC、SSID/CD 等概念之间的关系。
- **待项目核实**：MMU-720AE 具体版本实现了哪些可选能力，项目是否启用 RME、DPT、MEC，以及请求实际经过哪些检查路径。

## 一句话总结

> **MMU 管 CPU 地址，SMMU 管 Device/DMA 地址，MMU S3 则在设备翻译之上加入面向 RME/CCA 的 PAS 保护和新安全能力；架构支持不等于项目已经实现或启用。**

## 相关记录

- [Configuration Table Walk 与 GPT Walk](02_Configuration_Table_Walk与GPT_Walk.md)
- [ARM RME 扩展](10_ARM_RME扩展概述.md)
- [SMMU 地址转译流程与 Translation Context](16_SMMU地址转译流程与Translation_Context.md)
- [DPT：为 Full ATS 补充 Device/VM 级物理权限检查](41_DPT为Full_ATS补充设备权限检查.md)
