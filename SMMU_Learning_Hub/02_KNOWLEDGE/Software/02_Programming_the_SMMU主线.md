# Programming the SMMU：SMMUv3 软件编程主线

> 重建日期：2026-09-15  
> 状态：正式知识文档；按培训材料重新整理，待后续测试验证掌握程度。  
> 唯一主线来源：`MMU.pdf` 的 **Programming the SMMU** 模块，PDF 第 42–57 页（课件页 21–52）。

## 结论先行

SMMUv3 的编程重点不是“给每个设备配置一组翻译寄存器”，而是由软件建立一套以内存为主体的控制结构：

```text
寄存器
  └─ 给出 Stream Table、CMDQ、EVTQ、PRIQ 的位置与规模，并控制使能/中断

Stream Table / Context Descriptor Table
  └─ 用 SID/SSID 选择 Stage 1、Stage 2 以及相应页表配置

Command / Event / PRI Queue
  └─ 软件向 SMMU 下达维护命令，SMMU 向软件报告事件和页面请求
```

因此，理解本模块要抓住两条主线：

1. **配置主线**：软件如何让一个设备请求找到正确的 STE、CD 和翻译表。
2. **运行主线**：配置或页表发生变化、翻译失败或设备请求缺页时，软件如何通过队列与 SMMU 协作。

---

## 1. 编程接口为什么以 Memory-based Structures 为中心

培训材料首先给出 SMMUv3 的总体设计：多数配置不直接堆放在寄存器中，而是由软件分配内存，SMMU 寄存器只记录这些结构的位置和大小。

| 层次 | 主要对象 | 软件在做什么 |
|---|---|---|
| 寄存器层 | IDR、CR、表基址、队列基址、PROD/CONS、IRQ、错误寄存器 | 发现能力、告诉 SMMU 去哪里取结构、启用功能、观察全局状态 |
| 配置层 | Stream Table、STE、CD Table、CD | 描述某个 Stream/Substream 使用哪种翻译和哪套上下文 |
| 映射层 | Stage 1 / Stage 2 Translation Table | 描述输入地址如何映射以及对应权限/内存属性 |
| 消息层 | CMDQ、EVTQ、PRIQ | 在软件与 SMMU 之间传递维护命令、事件和页面请求 |

这种分工使配置规模能够随 Stream、Substream 的数量扩展。寄存器负责“入口与控制”，大批量、可扩展的 per-stream/per-process 状态放在内存中。

### 1.1 寄存器空间的安全状态划分

培训材料展示了两个连续的 64KB 主寄存器页，并说明 Secure 与 Non-secure 软件使用各自的寄存器视图。Root/Realm 控制页只有在实现 RME 时才出现：

- Root Control Page：由 Root PAS 访问；
- Realm Register Pages：仅在实现 RME 时存在；
- 这些可选页的存在和地址由具体实现决定。

这里应得出的结论是“寄存器访问受安全状态和实现能力约束”，而不是从示意图推断当前芯片必然支持 RME。

---

<a id="configuration-lookup"></a>

## 2. 从 SID/SSID 到 Translation Context

### 2.1 StreamID 先选择 STE

设备事务进入 SMMU 时携带 StreamID（SID）。SID 用于索引 Stream Table，得到对应的 Stream Table Entry（STE）。

培训材料把 STE 的作用概括为：

- 该 Stream 是无效、Bypass、Stage 1、Stage 2，还是 Stage 1 + Stage 2；
- Stage 2 的配置和页表入口；
- 如果启用 Stage 1，Context Descriptor Table 的入口与组织方式；
- Substream、ATS 等与该 Stream 相关的控制信息。

STE 不是地址映射本身。它先回答“这条 Stream 应当进入哪种翻译环境”。

### 2.2 Stream Table 可以是 Linear 或 2-Level

培训材料给出两种组织方式：

- **Linear Stream Table**：SID 直接选择连续表中的 STE，关系简单，但稀疏 SID 空间可能浪费内存。
- **2-Level Stream Table**：SID 的一部分索引一级表，一级描述符再指向二级表，最终选择 STE，适合较大的稀疏 SID 空间。

两级 Stream Table 只是配置表的存储组织，不是 Stage 1 + Stage 2 翻译，也不是多级 Translation Table Walk。

### 2.3 SSID 再选择 CD

当一个 Stream 需要承载多个地址空间时，事务还可以携带 SubstreamID（SSID）。STE 指向 CD Table，SSID 在该表中选择 Context Descriptor（CD）。

CD 保存 Stage 1 所需的上下文，例如：

- TTB0/TTB1：Stage 1 Translation Table 的入口；
- ASID：Stage 1 地址空间标识；
- T0SZ/T1SZ、TG0/TG1：地址范围和翻译粒度；
- IR/OR/SH：SMMU 访问页表内存时使用的属性；
- MAIR：解释页表描述符中内存属性索引的配置。

SID、SSID、ASID 的职责不能互换：

| 标识 | 出现位置 | 选择对象 |
|---|---|---|
| SID | 设备事务 | Stream / STE |
| SSID | 设备事务（启用 Substream 时） | 同一 Stream 下的 CD |
| ASID | CD 中的 Stage 1 配置 | 标识 Stage 1 地址空间，并参与相关翻译缓存管理 |

设备事务不直接携带操作系统的 ASID。SVA 场景中，软件通过绑定关系让设备使用的 SSID 找到包含相应 ASID 和 TTBR 的 CD。

### 2.4 Configuration Lookup 与 Translation Lookup 是两步

培训材料的完整逻辑是：

```text
输入事务：SID + 可选 SSID + 输入地址
        │
        ├─ Configuration Lookup
        │    SID  → STE
        │    SSID → CD（若使用 Stage 1 Substream）
        │    得到 StreamWorld、ASID、VMID、页表入口等上下文
        │
        └─ Translation Lookup
             先查 SMMU 的翻译缓存
             未命中时按所选上下文进行 Translation Table Walk
             输出地址以及访问权限等翻译结果
```

关键边界：STE/CD 决定“用什么规则和哪套表”，Translation Table Entry 决定“这个地址具体映射到哪里”。

---

## 3. 软件必须理解的三种 Queue

三种队列都是位于内存中的环形队列，但方向和语义不同。

| Queue | Producer | Consumer | 主要用途 |
|---|---|---|---|
| CMDQ | 软件 | SMMU | 软件提交配置缓存失效、TLB 失效、同步、PRI 响应等命令 |
| EVTQ | SMMU | 软件 | SMMU 报告与输入事务相关的配置、翻译或访问事件 |
| PRIQ（可选） | SMMU | 软件 | 将设备发来的 PCIe PRI Page Request 交给软件处理 |

BASE 指向队列存储区；PROD 表示生产进度；CONS 表示消费进度。谁是 Producer，谁就更新对应的 PROD；谁是 Consumer，谁就更新对应的 CONS。

### 3.1 CMDQ：软件控制 SMMU 的异步入口

基本提交关系是：

```text
软件确认有空位
  → 在 CMDQ 内存中写入命令
  → 让命令内容对 SMMU 可见
  → 更新 CMDQ_PROD
  → SMMU 读取并处理命令
  → SMMU 更新 CMDQ_CONS
```

培训材料重点介绍三类命令：

- **TLBI**：使指定范围的翻译缓存项失效；
- **CFGI**：使指定范围的 STE/CD 配置缓存项失效；
- **CMD_SYNC**：为同一 CMDQ 中此前命令提供同步完成点。

普通命令被 CMDQ_CONS 越过，首先表示该命令已经被消费，不能直接把它等同于所有相关效果已经对软件可见。需要完成边界时，应在相应命令之后安排 CMD_SYNC，并等待所选择的 SYNC 完成方式。

<a id="maintenance"></a>

### 3.2 更新内存结构后为什么还要维护缓存

SMMU 可以缓存：

- STE；
- CD；
- 地址翻译结果。

所以软件修改内存中的结构，不等于 SMMU 下一笔请求就一定读取到新内容。培训材料给出的维护对应关系是：

| 软件修改对象 | 需要处理的 SMMU 缓存 | 培训中对应的维护方向 |
|---|---|---|
| STE / CD | Configuration Cache | 通过 CMDQ 提交相应 CFGI |
| Translation Table 映射 | Translation Cache | 通过 CMDQ 或广播机制发出相应 TLBI |
| 需要确认此前维护已完成 | 前述命令的完成状态 | 在同一 CMDQ 中追加 CMD_SYNC |

培训示例展示了 `CMD_CFGI_STE` 后接 `CMD_SYNC`，由 SYNC 触发完成通知。

这是一条“修改对象—缓存类型—维护命令”的选择主线，不是完整的活动映射替换算法。培训本节没有展开 BBM、CPU 屏障、设备停流、ATC 失效以及资源回收的全部顺序，不能据此自行补成通用安全模板。

### 3.3 EVTQ：事务事件的报告路径

SMMU 将事件写入 EVTQ 并推进 EVTQ_PROD；软件读取条目、完成处理后推进 EVTQ_CONS。培训材料强调：

- 队列溢出会丢失记录；
- 可以在写入新事件时产生中断；
- 同一 Stream 的事件可能合并，STE.MEV 可控制相关行为。

EVTQ 用来保存具体事务事件；它不是软件向 SMMU 下命令的路径，也不等同于全局错误寄存器。

### 3.4 Global Error：影响 SMMU 全局运行的错误

培训材料把下列问题放在 Global Error 路径中：

- CMDQ 错误，且可能使后续命令处理暂停，直到软件清除错误；
- 访问 EVTQ 时发生 External Abort；
- 写 MSI 时发生 External Abort。

软件通过 `SMMU_(S_)GERROR` 观察错误状态，通过 `SMMU_(S_)GERRORN` 的规定方式确认/清除，并可配置 Global Error 中断。

运行时不能只盯着某个队列指针：CMDQ 停止、事件队列异常或 MSI 写失败都需要结合 Global Error 状态排查。

### 3.5 PRIQ：页面请求不是页面本身

在支持 PCIe ATS/PRI 的场景中，设备的 ATS Translation Request 可能因为页面尚未准备好而失败，随后设备发送 PRI Request。培训材料给出的软件闭环是：

```text
设备发送 PRI Request
  → SMMU 写入 PRIQ，并推进 PRIQ_PROD
  → 软件读取请求并准备页面
  → 软件推进 PRIQ_CONS，释放队列槽位
  → 软件通过 CMDQ 提交 CMD_PRI_RESP
  → Endpoint 得到成功或失败响应
```

推进 PRIQ_CONS 只代表软件消费了队列条目，不能替代 `CMD_PRI_RESP`。SMMU 也不会仅凭 PRIQ 条目自动分配或填充操作系统页面。

---

## 4. 从零启动 SMMU 的培训主线

培训附录给出的 Minimum Configuration Checklist 可以整理成以下依赖顺序：

1. **准备 Stream Table**  
   分配并初始化表内存，至少要正确初始化 Valid 状态；配置 `SMMU_STRTAB_BASE_CFG` 和 `SMMU_STRTAB_BASE`。

2. **准备 CMDQ 与 EVTQ**  
   分配并初始化队列内存；配置各自的 BASE、PROD、CONS。PRIQ 只有在实现并启用 PRI 时才需要。

3. **配置 SMMU 访问这些内存结构时的属性**  
   通过 `SMMU_CR1` 设置 SMMU 访问 Stream Table 和队列内存所需的 Cacheability/Shareability 等属性。

4. **配置中断路径**  
   根据实现和所启用功能准备 Event、PRI、Global Error 中断。

5. **设置 Global Bypass Attribute**  
   培训清单要求初始化 `SMMU_GBPA`，用于规定全局 Bypass 时的输出属性。

6. **启用队列与 SMMU**  
   培训清单列出 `SMMU_CR0.CMDQEN`、`EVENTQEN` 和 `SMMUEN`。

这份清单表达的是“最少需要准备哪些对象”，不是可以直接复制到驱动中的完整寄存器时序。正式实现仍需检查 IDR 能力、寄存器 ACK、内存可见性、错误处理和具体产品约束。

---

## 5. SVA 把前面的配置与队列串成闭环

培训最后用 Shared Virtual Addressing（SVA）展示前面各对象如何组合。

### 5.1 Bind 阶段

Linux 驱动把进程的地址空间与设备绑定：

- CD.TTBR 指向进程使用的 CPU Stage 1 页表；
- CD.ASID 使用该进程的地址空间标识；
- 为应用与设备绑定分配 SSID/PASID；
- 设备事务携带 SSID，SMMU 用它选择对应 CD；
- 驱动通过 `mm_notifier` 等机制加入 CPU 页表变化后的 SMMU TLB 维护链路。

这解释了为什么 SVA 不能只说“CPU 和设备共用同一个 VA”：软件还必须建立 SSID → CD → TTBR/ASID 的选择关系，并维护两侧缓存一致性。

### 5.2 缺页恢复阶段

培训时序图给出的主线是：

```text
应用启动设备工作
  → 设备发送 ATS Request
  → 页面尚未准备好，ATS 返回 translation failure
  → 设备发送 PRI Request
  → SMMU 把请求放入 PRIQ并触发软件处理
  → I/O page-fault handler 请求内存管理子系统处理缺页
  → CPU 页表被更新
  → 软件提交 CMD_PRI_RESP(success)
  → 设备重新发送 ATS Request
  → ATS translation success，设备缓存翻译
  → 设备发送 pre-translated transaction
```

这里的关键是职责划分：设备提出页面需求，SMMU 转交请求，操作系统建立映射，软件明确回复，设备随后重试翻译。

---

## 6. 附录能力在主线中的位置

培训附录还介绍了以下能力，它们不应被误认为所有实现的必选项：

### 6.1 STE/CD 字段速查

- STE 中的 Stage 1 相关字段用于描述 CD Table 及其访问属性；
- STE 中的 Stage 2 相关字段直接描述 IPA 范围、起始层级、粒度、PA 大小和 Stage 2 Table Walk 属性；
- CD 中的字段承担类似 CPU Stage 1 翻译控制寄存器、TTBR、MAIR 的职责。

字段表的作用是帮助定位配置归属。具体位编码、RES0/RES1、合法组合和更新约束仍应查 SMMUv3 Architecture Specification，不能只凭培训附录编程。

### 6.2 Embedded Configuration

架构允许“指向内存结构的寄存器”为只读；某些实现中，对应地址后面可以是实现定义的寄存器式配置而非普通内存。这是一种实现选择，不改变“软件通过架构定义入口让 SMMU 获得配置”的总体关系。

### 6.3 ATOS / VATOS

ATOS 是可选的软件查询翻译接口：软件提供输入地址、SID 和翻译类型，启动操作，等待完成，再从结果寄存器读取翻译结果。VATOS 是可选的虚拟化形式，可限制 VM 只查询自身范围。

是否存在 ATOS/VATOS 必须先查 `SMMU_IDR0.ATOS` 及具体产品文档，不能因培训介绍过就视为当前实现具备。

---

<a id="source-boundary"></a>

## 7. 培训材料的使用边界与需核验处

为了避免再次把课件简化内容写成过度确定的架构结论，本篇保留以下边界：

1. **页码边界**  
   本篇只整理 PDF 第 42–57 页。旧文档中来自第 32–41、58 页以后以及 MMU-720AE TRM 的内容，不再混入本篇主线。

2. **CMDQ 错误报告的课件表述需要架构核验**  
   培训第 49–50 页把 `CERROR_ILL`、EVTQ 与 CMDQ 错误原因联系起来；但当前资料库已有校正指出，普通 CMDQ 错误的架构排查主路径是 `GERROR.CMDQ_ERR` 与 `CMDQ_CONS.ERR`。因此本篇只保留“CMDQ 错误属于 Global Error，可能暂停命令处理”这一稳定主线，不把课件示例提升为通用规则。

3. **寄存器名称以架构规范为准**  
   培训 Minimum Configuration 页使用了 `SMMU_EVENT_*` 写法；架构常用名称为 `SMMU_EVTQ_*`。正文使用逻辑对象名，不把课件拼写当作最终寄存器定义。

4. **最小配置清单不是完整驱动时序**  
   培训没有在该页展开能力发现、ACK 等待、写入可见性和失败回滚。清单只能用于建立对象依赖，不可直接当作生产代码。

5. **SVA 图是机制主线，不是完整 Linux API 规范**  
   图中函数名和驱动关系用于说明 TTBR/ASID/SSID、ATS、PRI、页故障处理如何连接；具体内核版本的 API 和锁/生命周期规则需要结合对应 Linux 源码核实。

---

## 8. 用一条流程复述整个模块

```text
软件发现 SMMU 能力和安全状态视图
  → 分配并初始化 Stream Table、CD Table、CMDQ、EVTQ、可选 PRIQ
  → 配置表/队列基址、规模、访问属性和中断
  → 用 STE 定义 Stream 的翻译模式，用 CD 定义 Stage 1 context
  → 启用队列和 SMMU
  → 请求到达后，以 SID/SSID 完成 Configuration Lookup
  → 使用选出的 context 查询翻译缓存或进行 Translation Table Walk
  → 配置/页表变化后，用 CFGI/TLBI 并在需要时以 CMD_SYNC 建立完成边界
  → 事务事件通过 EVTQ 报告，全局运行错误通过 GERROR 报告
  → ATS 缺页时，经 PRIQ、软件缺页处理和 CMD_PRI_RESP 完成恢复
```

## 面试版总结

> SMMUv3 采用 memory-based programming model：寄存器给出表和队列的入口，SID/SSID 通过 STE/CD 选择翻译上下文，CMDQ 负责软件维护，EVTQ 和 PRIQ 把事件与页面请求交给软件；配置更新后还必须处理 SMMU 对 STE、CD 和翻译结果的缓存。

## 来源

- [MMU.pdf：Programming the SMMU](../../99_SOURCE/Training/MMU.pdf)，PDF 第 42–57 页。

