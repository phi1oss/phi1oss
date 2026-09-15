# DTI 协议、连接与 Token 流控

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 81–82、85–88（不是单张 slide 编号）。

## 核心结论与设计目的

DTI 定义翻译控制消息；BAS 承载消息；Token 限制可在途的服务请求。物理链路接通、TVALID/TREADY 握手成功、DTI 连接建立、翻译完成，是不同层面的状态。

## 1. 协议分层与版本

培训第 81/86 页把消息分为连接、翻译、失效同步、页面请求、寄存器访问五类。DTI-TBU 服务 TBU↔TCU，DTI-ATS 服务 Root Port↔TCU。

培训通用 DTI 页仍使用 AXI4-Stream 名称，MMU S3 接口模块及 MMU-720AE TRM 明确使用 AXI5-Stream。对当前产品采用后者，不用通用历史 slide 覆盖产品接口定义。

MMU S3 r0/MMU-720AE 的内部 DTI 对应 v3；S3 r1 的 v4 不能直接套用。外部 ATS 或 built-in TBU 支持集合须逐产品确认。

## 2. 连接角色与消息方向

培训第 88 页及 TRM 第 46 页：TBU/PCIe Root Port 是 DTI requester（培训称 manager），TCU 是 completer（subordinate）。连接由前者发起，TCU 可以接受或拒绝。

这不是“所有消息必须由 manager 发请求”的规则：失效和 TBU 寄存器访问请求可由 TCU 发起。BAS switch 的 S/M 端口名称则是互连端口视角，不能把整块 TCU 改称 DTI manager。

## 3. Token 管容量

| 类型 | 授予者 | 受限请求方向 | 回收 |
|---|---|---|---|
| Translation token | TCU | TBU/Root Port → TCU | 相应 response 返回 |
| Invalidation token | TBU/Root Port | TCU → requester | 相应 ack 返回 |

没有足够 token，不能发对应请求。Link-level ready 表示传输接收条件，不能替代 translation/invalidation 的资源授权。TID 则负责身份，不负责提供资源。

“所有请求都消耗同一 token”是错误概括；不同消息类别必须看各自协议规则。

## 4. MMU-720AE 的具体约束

TRM §2.5.1：max_tok_trans 指定请求的 translation token 数；每个 LTI port 至少需要两个 translation tokens。连接中的 TOK_TRANS_REQ 请求资源，TOK_INV_GNT 授予失效资源。

该 TBU 只授予一个 invalidation token，因此在此前 ack 归还 token 前，不能直接套用“连续发两条 INV_REQ”的多 token 示意。已归还 token 不等于失效所有效果已完成，仍需 SYNC 语义。

培训第 82 页还要求请求 token 数不超过 TCU translation slots。项目配置需同时满足这些约束，不能凭示例设任意数量。

## 5. 生命周期与自测

Reset 后连接 → 协商/分配资源 → 在 token 限额内工作 → 请求断连并归还资源 → 断连完成。电源关断还须满足低功耗流程，不由 CONDIS 单独包办。

自测：TREADY=1 但 token=0 能否发对应翻译请求？为何 INV_ACK 可以回收容量，却不能证明失效完成？

一句话：DTI 定义服务消息，连接建立服务状态，Token 控制容量，TID 区分身份。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 81–82、85–88 页；文内“培训依据”均指上述页面。
- 产品依据：[MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)，§2.4.1.6 第 35 页、§2.5.1 第 45–46 页。
- 来源问答：[52_DTI协议分层_消息类型与完整流程](../../01_INBOX/ChatGPT/52_DTI协议分层_消息类型与完整流程.md)
- 来源问答：[53_DTI协议概览与Token流控](../../01_INBOX/ChatGPT/53_DTI协议概览与Token流控.md)
- 来源问答：[54_DTI连接管理与Token生命周期](../../01_INBOX/ChatGPT/54_DTI连接管理与Token生命周期.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
