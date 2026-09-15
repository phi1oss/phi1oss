# SVA 绑定、PRI 与 DTI 页面恢复

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 46、50–51、55–57、91（不是单张 slide 编号）。

## 核心结论与设计目的

SVA 解决“设备使用谁的进程地址空间”，ATS 解决“提前取得翻译”，PRI 解决“页面当前不可用时请求软件处理”。三者常配合，但不是同一机制，也不应把 SVA 无条件等同于必须采用 ATS/PRI 的所有实现。

## 1. SVA Bind 建立什么关系

培训第 55–57 页的实例把进程页表根与 ASID 放进对应 CD，通过设备支持的 PASID/SSID 选择它：

```text
设备工作归属 → SID + SSID → STE / CD → 进程页表 + ASID
```

仅传 VA 不足以区分两个进程的同名地址。绑定由系统软件建立，不是 SMMU 从地址自动猜出进程。页表改变时还需要通知并维护设备/SMMU 缓存；解除绑定也涉及请求生命周期。

讲义中的 Linux 函数名是其示例版本背景，本篇不将其视为当前内核 API 保证。

## 2. PRI 的完整闭环

```text
ATS 查询失败（页面未准备好等）
→ Endpoint 发 PRI 请求
→ Root Port 用 DTI_ATS_PAGE_REQ 交给 TCU
→ TCU 将请求交给 PRIQ
→ 软件读取、决定是否及如何准备页面
→ 软件通过 CMDQ 发 CMD_PRI_RESP
→ TCU 用 DTI_ATS_PAGE_RESP 返回处理结果
→ 设备依据成功/失败结果继续或重试
```

页面修复可能涉及分配、映射、权限与必要维护，不是硬件保证必定成功的动作。

## 3. 四种 PAGE 消息

| 消息 | 方向 | 意义 |
|---|---|---|
| PAGE_REQ | Root Port → TCU | 递交 Page Request |
| PAGE_ACK | TCU → Root Port | 请求接收确认 |
| PAGE_RESP | TCU → Root Port | 软件处理结果 |
| PAGE_RESPACK | Root Port → TCU | 响应接收确认 |

第 91 页着重描述成功准备页面的场景，第 51 页明确 CMD_PRI_RESP 可报告成功或失败。因此不能把任何 PAGE_RESP 都解释为“页面已可用”。

PRIQ_CONS 前进不替代 CMD_PRI_RESP；PAGE_ACK 也不是页面修复完成。架构第 242 页进一步指出 CMD_SYNC 不能为 CMD_PRI_RESP 保证 Endpoint 可见性。

## 4. 与 Stall/Resume 的区别

Stall 是 SMMU 暂挂原事务后的恢复；PRI 是设备请求软件准备页面，再按响应决定下一步。没有 PRI 时，系统可能提前准备/固定页面，或在适用请求上另用 Stall，但后者不是对 PRI 协议的透明替代。

## 5. 未解决问题与自测

PRG 分组、最后请求标志、失败编码、超时及设备重试规则仍需 DTI/PCIe PRI 规范与驱动实例。当前没有证据表明项目已启用 SVA/PRI，文档描述的是机制。

自测：为什么 PRIQ_CONS、PAGE_ACK、PAGE_RESP 分别不能互相替代？页面准备失败时系统如何避免永远等待？

一句话：SVA 绑定地址空间，ATS 查询翻译，PRI 将缺页处理交给软件；握手确认不等于页面可用。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 46、50–51、55–57、91 页；文内“培训依据”均指上述页面。
- 架构补充：[SMMUv3 Architecture Specification](../../99_SOURCE/Architecture_Spec/System_Memory_Management_Unit_Architecture_Specification.pdf)，§4.7.3 第 242 页，CMD_PRI_RESP 的可见性边界。
- 来源问答：[19_Substream_SSID与PASID](../../01_INBOX/ChatGPT/19_Substream_SSID与PASID.md)
- 来源问答：[28_Substream如何关联进程地址空间](../../01_INBOX/ChatGPT/28_Substream如何关联进程地址空间.md)
- 来源问答：[37_PRI缺页恢复与无PRI退化路径](../../01_INBOX/ChatGPT/37_PRI缺页恢复与无PRI退化路径.md)
- 来源问答：[38_SVA共享虚拟地址与进程设备绑定](../../01_INBOX/ChatGPT/38_SVA共享虚拟地址与进程设备绑定.md)
- 来源问答：[59_DTI-ATS_Page_Request与PRI恢复闭环](../../01_INBOX/ChatGPT/59_DTI-ATS_Page_Request与PRI恢复闭环.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
