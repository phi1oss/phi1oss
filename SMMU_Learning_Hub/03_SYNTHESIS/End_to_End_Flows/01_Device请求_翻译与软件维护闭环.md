# Device 请求、翻译与软件维护闭环

> 更新日期：2026-09-10  
> 状态：跨主题综合；基于本轮正式文档建立逻辑联系，待脱稿讲解验证。

## 1. 先分三条路径

```text
数据路径：Device → ACE-Lite TBU / 适用的 Lookaside 数据路径 → System
翻译路径：TBU miss / ATS query → DTI over BAS → TCU → 结果返回
软件路径：写 STE/CD/PTE 与 CMDQ → SMMU 维护 → EVTQ/PRIQ/完成通知
```

这不是强制每个系统有相同接线。Inline/Cached/Lookaside 决定真实数据经过哪里；DTI 不携带 DMA payload。

## 2. 一笔普通设备访问

1. 请求携带地址、访问属性和适用 SID/SSID/安全状态。
2. 逻辑上确定 Context：SID→STE；需要时 SSID→CD。缓存命中可以避免内存读取。
3. TBU 若已有适用结果，按结果及权限继续；需要 TCU 时，在连接与 token 条件满足后发 DTI。
4. BAS 用正确 ID/目的路由传消息；TCU 按 Context 查询翻译，必要时 walk。
5. RESP/RESPEX 返回地址和属性；FAULT 返回失败信息。用 TID/TDEST 匹配，不要求所有响应按请求次序返回。
6. 数据访问按模式继续，或按支持的 fault model 终止/暂挂。

此链描述依赖关系，不规定所有权限/GPC 检查必须最后串行发生。

## 3. 映射改变后的软件维护

```text
正确更新结构并满足可见性/BBM等前提
→ 对适用对象发 CFGI / TLBI / ATC invalidation
→ 必要的分布式失效及 DTI 同步
→ 目标 CMD_SYNC 完成
→ 在设备生命周期等其他条件也满足后，执行依赖完成的后续动作
```

CFGI 不修改 PTE，普通 INV_ACK 不保证所有失效效果完成。目标 CMD_SYNC 被正常消费则具备其架构完成保证；不等于全芯片或全部 DMA 都已停止。

## 4. 页面未准备好时的两条不同恢复路

- 普通适用请求：Stall → EVTQ → 软件修复 → Resume/终止。
- ATS/PRI：翻译失败 → Page Request → PRIQ → 软件处理 → CMD_PRI_RESP → 页面处理结果 → 设备依据结果重试或失败处理。

不得把 PAGE_ACK、PRIQ_CONS 或 CMD_SYNC 当成 Endpoint 已收到成功 Page Response 的证明。

## 来源与导航

- [Context](../../02_KNOWLEDGE/Software/02_Programming_the_SMMU主线.md#configuration-lookup)
- [拓扑](../../02_KNOWLEDGE/Architecture/01_MMU_S3组件与设备集成拓扑.md)
- [维护与同步](../../02_KNOWLEDGE/Software/02_Programming_the_SMMU主线.md#maintenance)
- [DTI 消息](../../02_KNOWLEDGE/Interfaces/07_DTI翻译响应_失效同步与寄存器访问.md)
- [PRI 恢复](../../02_KNOWLEDGE/Interfaces/02_SVA绑定_PRI与DTI页面恢复.md)

训练任务：不看资料讲完上述三条路径，并指出至少三个不能混同的完成点。
