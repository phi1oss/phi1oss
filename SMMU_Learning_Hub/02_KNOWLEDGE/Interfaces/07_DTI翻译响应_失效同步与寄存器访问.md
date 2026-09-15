# DTI 翻译响应、失效同步与寄存器访问

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 88–91（不是单张 slide 编号）。

## 核心结论与设计目的

DTI 中的“返回消息”不具有统一完成含义：翻译响应返回结果，INV_ACK 回收流控资源，SYNC_ACK 确认先前失效完成，REG_WACK 完成特定寄存器写。复盘时必须先问它确认的是哪一件事。

## 1. 翻译请求与结果

第 88 页：

| 消息 | 主要内容 | 意义 |
|---|---|---|
| DTI_TBU_TRANS_REQ | IA、SID、SSID、SEC_SID、访问属性 | 请求在相应 Context 下翻译/检查 |
| DTI_TBU_TRANS_RESP | OA、ASID/VMID、属性 | 返回翻译或 Bypass 结果 |
| DTI_TBU_TRANS_FAULT | fault 类型、是否可缓存等 | 翻译未能合法完成 |

TBU 本地 TLB miss 是典型触发条件；也可有 speculative request。第 88 页时序图显示响应次序可以与发出次序不同，靠 unique-in-flight TID/TDEST 匹配；不能把 token 当返回身份。

不是所有 fault 都能缓存，也不能把 speculative request 简化为“必定触发 TCU 继续预取”。精确 prefetch 规则留到 Caching 模块。

## 2. RESPEX 与 MECID

第 89 页明确：

- DTI_TBU_TRANS_RESPEX 在普通结果之外增加 MECID，适用于列出的 v3/v4。
- REQEX=1 时可以返回 RESPEX，但 TCU 仍可返回普通 RESP 或 FAULT。
- 普通 RESP 等价于扩展响应中 MECID 全零的情形。
- 扩展消息为 192 bit，对比普通 160 bit；互连需能承载。

不能从 REQEX=1 推导“肯定加密”或“肯定收到扩展响应”；MECID 的系统意义还取决于配置。

## 3. INV 与 SYNC

```text
TCU → INV_REQ → TBU/Root Port
TCU ← INV_ACK ← requester       （流控确认）
TCU → SYNC_REQ → requester
TCU ← SYNC_ACK ← requester      （先前相关失效完成）
```

第 90 页列举触发来源：DVM TLBI/Sync、CMDQ 维护命令、特定寄存器变化的隐式失效。DVM TLBI 不用于配置缓存失效。

DTI-TBU 与 DTI-ATS 不应机械共享全部失效字段和对象；讲义把两者合列是概述，本篇不扩展成“所有 requester 都保存 STE/CD”。

针对 MMU-720AE TBU，TRM 规定一个 invalidation token，故示意中的多 INV 并发须服从实际 token 额度。

## 4. TCU 访问远端 TBU 寄存器

第 90 页下半：

- TCU 发 REG_READ(ADDR, NS)，TBU 返回 REG_RDATA。
- TCU 发 REG_WRITE(ADDR, NS, DATA)，TBU 返回 REG_WACK。
- 用于 TBU u-Arch、RAS、MPAM 等寄存器，只适用于 TBU。

DTI requester 角色不限制 TCU 发起这类请求；方向由消息功能决定。地址映射、访问错误和更精确的完成约束仍需协议/产品寄存器说明。

## 5. Page Request 与自测

第 91 页的 PAGE_REQ/ACK/RESP/RESPACK 连接到 PRIQ 与软件响应，详见 [SVA 与 PRI](02_SVA绑定_PRI与DTI页面恢复.md)。PAGE_ACK 不能代表软件修复页面，PAGE_RESP 也必须区分成功或失败结果。

自测：为什么 INV_ACK 不能替代 SYNC_ACK？REQEX=1 收到普通 RESP 应怎样理解？为什么 TCU 可以主动发寄存器访问？

一句话：REQ 请求服务，RESP 返回翻译，INV_ACK 归还容量，SYNC_ACK 确认失效，REG 消息代理 TBU 寄存器访问。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 88–91 页；文内“培训依据”均指上述页面。
- 产品依据：[MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)，§2.5.1 第 46 页，invalidation token 约束。
- 来源问答：[52_DTI协议分层_消息类型与完整流程](../../01_INBOX/ChatGPT/52_DTI协议分层_消息类型与完整流程.md)
- 来源问答：[56_DTI翻译请求_响应与MECID扩展](../../01_INBOX/ChatGPT/56_DTI翻译请求_响应与MECID扩展.md)
- 来源问答：[57_DTI失效与同步_INV_ACK和SYNC_ACK](../../01_INBOX/ChatGPT/57_DTI失效与同步_INV_ACK和SYNC_ACK.md)
- 来源问答：[58_DTI远程访问TBU寄存器](../../01_INBOX/ChatGPT/58_DTI远程访问TBU寄存器.md)
- 来源问答：[59_DTI-ATS_Page_Request与PRI恢复闭环](../../01_INBOX/ChatGPT/59_DTI-ATS_Page_Request与PRI恢复闭环.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
