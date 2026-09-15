# GitHub 手机端记录人工导入日志

本文件用于记录“手机端 GitHub 时间段”的内容何时、以何种方式整理进本地知识库。GitHub 与本地不是自动同步关系；只有用户明确要求时才进行人工比对、提取和导入。

## 2026-09-03

来源仓库：[phi1oss/phi1oss：SMMU 学习资料库](https://github.com/phi1oss/phi1oss/tree/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93)

| GitHub 手机端记录 | 本地处理结果 | 说明 |
| --- | --- | --- |
| `2026-09-03_1007_PRI_Queue与无PRI时ATS缺页处理.md` | [Inbox 37](../01_INBOX/ChatGPT/37_PRI缺页恢复与无PRI退化路径.md) | 保留完整 PRI 恢复闭环，重点提取“无 PRI 时如何退化”，并关联既有三 Queue、ATS 和 Stall 记录 |
| `2026-09-03_1110_SVA共享虚拟地址与进程设备绑定.md` | [Inbox 38](../01_INBOX/ChatGPT/38_SVA共享虚拟地址与进程设备绑定.md) | 单独建立 SVA 主题，关联 SID/SSID、STE/CD、ATS、PRI 与 Stage 2 |
| `SMMUv3_Stream_Table与CPU页表上下文__2026-09-01.md` | 未重复建档 | 内容已由本地 Inbox 23、25–33 分主题覆盖；仅完成比对，不复制重复正文 |

## 2026-09-06（导入 GitHub 9 月 4 日记录）

来源仓库：[phi1oss/phi1oss：SMMU 学习资料库](https://github.com/phi1oss/phi1oss/tree/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93)

| GitHub 手机端记录 | 本地处理结果 | 说明 |
| --- | --- | --- |
| `2026-09-04_1532_MMU_S3系统定位与SMMU基础.md` | [Inbox 43](../01_INBOX/ChatGPT/43_MMU_S3系统定位与能力分层.md) | 保留 MMU 三项职责、CPU MMU/SMMU 分工和 MMU S3 的 RME/CCA 能力层次，并标注架构支持与项目实现边界 |
| `2026-09-04_1533_PCIe_Root_Complex系统定位与SMMU关系.md` | [Inbox 44](../01_INBOX/ChatGPT/44_PCIe_Root_Complex_Root_Port与Bifurcation.md) | 与 Root Port、bifurcation 合并，形成 PCIe 主机侧组件、身份映射和 SMMU 接入主题 |
| `2026-09-04_1551_PCIe_bifurcation概念.md` | [Inbox 44](../01_INBOX/ChatGPT/44_PCIe_Root_Complex_Root_Port与Bifurcation.md) | 与 Root Complex/Root Port 合并，避免把同一系统关系拆成三个零散名词文件 |
| `2026-09-04_1552_PCIe_Root_Port概念.md` | [Inbox 44](../01_INBOX/ChatGPT/44_PCIe_Root_Complex_Root_Port与Bifurcation.md) | 与 Root Complex/bifurcation 合并，并保留 RC 与 RP 的职责边界 |
| `2026-09-04_1534_DMA机制与SoC典型应用.md` | [Inbox 45](../01_INBOX/ChatGPT/45_DMA机制_SoC应用与SMMU隔离.md) | 独立建立 DMA 基础主题，补足 requester、SMMU 隔离、coherency 与 fault 集成检查项 |
| `2026-09-04_1626_PCIe_PASID概念.md` | 未重复建档；关联 Inbox [19](../01_INBOX/ChatGPT/19_Substream_SSID与PASID.md)、[28](../01_INBOX/ChatGPT/28_Substream如何关联进程地址空间.md)、[38](../01_INBOX/ChatGPT/38_SVA共享虚拟地址与进程设备绑定.md) | 本地既有记录已覆盖 Requester ID/PASID、SID/SSID、STE/CD、SVA/ATS/PRI 关系；完成内容比对，不重复复制 |
| `2026-09-04_1719_LPD-500低功耗分发与Sequencer行为.md` | [Inbox 46](../01_INBOX/ChatGPT/46_LPD-500低功耗分发与Sequencer.md) | 独立建立低功耗主题，区分 group、Expander、Sequencer、物理关断和 FuSa 边界 |

## 2026-09-07（导入 GitHub 9 月 7 日记录）

来源仓库：[phi1oss/phi1oss：SMMU 学习资料库](https://github.com/phi1oss/phi1oss/tree/main/SMMU%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99%E5%BA%93)

| GitHub 手机端记录 | 本地处理结果 | 说明 |
| --- | --- | --- |
| `AMBA_Atomic_Transaction__2026-09-07.md` | [Inbox 50](../01_INBOX/ChatGPT/50_AMBA_Atomic_Transaction与Exclusive_Access.md) | 48 号保留 Property 总览；本条独立保存 Atomic 竞争模型、AWATOP/operand、与 Exclusive Access 的差异以及 TBU 职责 |

## 使用约定

- GitHub 保留手机端原记录，本地保留整理后的编号记录。
- 人工导入时按知识边界拆分、合并和去重，不机械复制目录结构。
- 不自动把本地内容反向上传 GitHub，也不自动合并两个时间段。
- 后续导入先查看本日志，再检查 GitHub 新增文件和本地最大编号。
