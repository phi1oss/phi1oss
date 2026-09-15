# SMMU 低功耗、复位与连接顺序

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 70、77、82、87–88（不是单张 slide 编号）。

## 核心结论与设计目的

时钟、电源与 DTI 连接状态必须协调。关闭一个 TBU，不只是拉低电源控制；必须让其协议连接和相关事务按要求收尾。TCU 是被多个 requester 共享的资源，关闭条件更严格。

## 1. 全局接口

培训第 70 页指出：组件有独立 clock/reset 输入；TBU/TCU 有内部 reset 同步，BAS 组件需要外部同步复位。不同 clock/power domain 可通过 ADB-400 BAS 桥连接。

Q-Channel 的 LPI_CG 用于 clock gating，TBU/TCU 的 LPI_PD 用于 power down/up。BAS switch 是讲义明确列出的 clock-gating LPI 例外，不能认为所有互连组件接口完全一致。

## 2. 电源顺序

培训第 77 页明确要求：

- TBU 断电前先与 TCU 断连。
- TCU 断电前，所有 TBU 与 PCIe Root Port 都必须断连。
- TCU 先上电，之后其他 requester 才能上电并连接。
- 相应 Q-Channel 必须到 Q_STOPPED，才可撤时钟或电源。
- power control 期间需要时钟活动，不能同时任意 clock gate。
- TBU/TCU 不保留断电前全部状态，上电后需重新编程 SMMU 寄存器。

这些是产品培训约束，不是可省略状态确认的固定延时脚本。

## 3. DTI 连接资源的生命周期

连接请求分配 translation tokens 并授予 invalidation tokens；断连时按协议收尾和归还资源。DTI 握手完成是低功耗流程的一部分，不等同于已经取得所有 Q-Channel/系统电源安全条件。

```text
停止/约束新工作 → 排空与 DTI 断连 → 满足低功耗握手条件 → 断电
上电及复位处理 → 恢复配置 → 建立 DTI 连接 → 恢复工作
```

这是学习用的依赖关系，具体事件的严格次序和可并行项须依据产品集成设计。

## 4. LPD-500 的归档边界

培训说明 LPD-500 可把单个 clock/power controller 接到多个组件。原问答 46 的 Expander/Sequencer、分组与拒绝传播细节尚无本地 LPD-500 规范核实，本篇不晋升为产品配置规则。

当前项目的分组、上电依赖、reset 撤销同步及 fault 时的关断路径仍待设计资料。

## 自测与面试一句话

自测：为何关闭 TCU 要先检查全部 requester？为什么 power control 期间反而需要保持 clock？

一句话：先让协议和事务安全退出，再满足低功耗握手；上电恢复则先准备 TCU 与配置，再恢复连接。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 70、77、82、87–88 页；文内“培训依据”均指上述页面。
- 来源问答：[46_LPD-500低功耗分发与Sequencer](../../01_INBOX/ChatGPT/46_LPD-500低功耗分发与Sequencer.md)
- 来源问答：[54_DTI连接管理与Token生命周期](../../01_INBOX/ChatGPT/54_DTI连接管理与Token生命周期.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
