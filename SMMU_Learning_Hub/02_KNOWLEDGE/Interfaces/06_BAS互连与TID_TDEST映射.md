# BAS 互连与 TID/TDEST 映射

> 更新日期：2026-09-10  
> 状态：正式知识文档；核心结论已对照所列资料，掌握程度待测试。  
> 主线：MMU 培训文档，PDF 文件页码 82–83、86（不是单张 slide 编号）。

## 核心结论与设计目的

BAS 为多个 requester 提供双向 DTI 传输。Switch 汇聚和返回路由，Sizer 改变传输宽度，Register Slice 分割长时序路径；跨时钟/电源域还需要合适的桥。

## 1. 组件组合

| 组件 | 解决的问题 | 不承担的职责 |
|---|---|---|
| Switch | 多端口汇聚与 ID/目的路由 | 不执行地址翻译 |
| Sizer | 带宽、传输拍数与布线宽度折中 | 不改变 DTI 消息语义 |
| Register Slice | 改善时序路径 | 不提供地址转换 |
| ADB-400 BAS | 跨 clock/power domain | 不是 switch 自带功能 |

TRM 第 31 页明确 MMU-720AE 互连组件本身不包含跨域桥；可另用 ADB-400。拓扑按系统需求组合，不是固定链路模板。

## 2. 两个方向

- Downstream：TBU/Root Port → TCU。
- Upstream：TCU → TBU/Root Port。

第 83 页的 switch 图中，DN_Sn 输入汇聚到 DN_M；UP_M 输入按目的解码后发往 UP_Sn。S/M 是这组端口的命名，DTI 的组件角色要另看协议定义。

## 3. ID 区间示例

培训配置表如下：

| S 口 | 本地 TID/TDEST | M 侧区间 |
|---|---|---|
| S0 | 0–5 | 0–5 |
| S1 | 0–3 | 6–9 |
| S2 | 0–3 | 10–13 |
| S3 | 0–7 | 14–21 |

DECMIN_SIn/DECMAX_SIn 指定各输入的映射区间边界。区间必须能唯一解码，不能相互重叠。

对该图示映射，S2 的本地 2 对应汇聚侧 12；返回 TDEST=12 落入 10–13 区间，送到 S2 并恢复本地 2。这里的“全局”仅指该汇聚命名空间，不是终身全 SoC 唯一事务号。

加减 DECMIN 是对示例映射的解释；多级 switch 的实际参数合法性、编码和命名空间规划仍需集成规范。

## 4. ID_WIDTH 与位宽

无符号 W 位能表示 0 至 2^W−1。因此，要容纳最大 ID=M，需要满足 2^W > M；该例 M=21，至少 5 位。讲义简写 log2(DECMAX) 时，不应忽略取整及 M 恰为 2 的幂的边界。

这是位数计算推导，不是另行核实过的 switch 参数全部约束。

培训第 82 页描述下行接口 160 bit，上行 160/192 bit 随 MECID 配置变化。经过 sizer 的内部链路可以更窄；消息宽度不能直接当作每条物理链路必有的线宽。

## 5. 自测与面试一句话

自测：S1 与 S2 的本地 TID 都为 2 时，汇聚后如何区分？最大 ID=16 需要多少位？为什么加 Register Slice 不能代替跨域桥？

一句话：BAS 传消息、Switch 管路由；请求扩大 ID 命名空间，响应按 TDEST 找回来源。

## 来源与相关记录

- 主资料：[MMU.pdf](../../99_SOURCE/Training/MMU.pdf)，PDF 第 82–83、86 页；文内“培训依据”均指上述页面。
- 产品依据：[MMU-720AE TRM](../../99_SOURCE/TRM/arm_corelink_mmu_720ae_system_memory_management_unit_technical_reference_manual_109745_0002_04_en.pdf)，第 31、35、45 页。
- 来源问答：[55_BAS互连组件与TID_TDEST路由](../../01_INBOX/ChatGPT/55_BAS互连组件与TID_TDEST路由.md)

本篇保留来源追溯，但不把全部原问答自动视为已验证结论；未纳入正文的细节仍留在来源层。
