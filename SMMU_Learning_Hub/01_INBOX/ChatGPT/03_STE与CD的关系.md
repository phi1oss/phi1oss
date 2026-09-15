# Stream Table Entry（STE）与 Context Descriptor（CD）

## 问题

在 SMMU 的 Configuration Table Walk 中，STE 和 CD 分别是什么？二者有什么关系？

## 核心结论

**STE 用 StreamID 确定“这条设备数据流如何翻译”，CD 在启用 Stage 1 时进一步用 SubstreamID/PASID 确定“该数据流中的具体进程使用哪个地址空间”。STE 管设备流，CD 管进程地址空间。**

## 1. STE：以设备数据流为粒度的入口配置

设备发出的事务通常携带 `StreamID`。SMMU 使用 StreamID 查询 Stream Table，并获得对应的 **Stream Table Entry（STE）**。

STE 负责描述这条 stream 的总体处理方式，例如：

- 事务是 bypass、abort，还是需要 translation。
- 是否启用 Stage 1、Stage 2，或者两级翻译。
- Stage-2 translation 的 VMID、页表基地址及相关控制信息。
- 启用 Stage 1 时，如何找到 Context Descriptor Table。

因此，STE 回答的核心问题是：

> **“这个设备数据流应该采用什么翻译模式？”**

## 2. CD：以进程或地址空间为粒度的 Stage-1 配置

如果某条 stream 启用了 Stage-1 translation，而且设备支持多个进程地址空间，请求中还可能携带 `SubstreamID/PASID`。

SMMU 根据 STE 提供的 Context Descriptor Table 配置，再利用 SubstreamID 查找相应的 **Context Descriptor（CD）**。

CD 主要描述 Stage-1 translation context，例如：

- Stage-1 页表基地址。
- ASID。
- 翻译粒度和输入地址范围。
- Stage-1 地址翻译及权限相关控制信息。

因此，CD 回答的核心问题是：

> **“这条设备流中的这个进程，应该使用哪个 Stage-1 地址空间？”**

## 3. STE 与 CD 是两级索引关系

二者不是并列、互相替代的配置项，而是从设备到进程逐级细化：

```text
设备事务
   │
   ├── StreamID
   │      ↓
   │   Stream Table
   │      ↓
   │     STE
   │   “设备流如何处理？”
   │      │
   │      ├── Stage 2 配置可直接来自 STE
   │      │
   │      └── 如果启用 Stage 1
   │              ↓
   └── SubstreamID / PASID
          ↓
      Context Descriptor Table
          ↓
         CD
      “进程使用哪个 Stage-1 地址空间？”
```

可以简化为：

```text
StreamID → STE → SubstreamID/PASID → CD
  设备流配置                 进程地址空间配置
```

## 4. 为什么需要把 STE 和 CD 分开？

一个设备可能服务多个进程。如果每个进程都复制一份完整的设备级配置，会造成大量重复。

将二者分开后：

- 同一设备流共享一份 STE，统一描述设备级策略和 Stage-2 环境。
- 不同进程分别使用不同 CD，保存各自的 Stage-1 页表和 ASID。

例如：

```text
同一个加速器：StreamID = 0x20
                 │
                 ▼
               STE
          共同的设备级配置
             /         \
PASID = 1 → CD1       CD2 ← PASID = 2
             │         │
        进程 A 页表   进程 B 页表
```

这样，同一设备便可以在隔离的地址空间中代表多个进程访问内存。

## 5. 它们在完整翻译流程中的位置

```text
Translation Request
StreamID + SubstreamID/PASID + IOVA
             ↓
根据 StreamID 查 STE
“确定设备流的翻译模式”
             ↓
需要 Stage 1 时，根据 SubstreamID 查 CD
“获得进程的 Stage-1 translation context”
             ↓
执行 Stage-1 Translation Table Walk
IOVA → IPA
             ↓
执行 Stage-2 Translation Table Walk（启用时）
IPA → PA
             ↓
GPT Walk / PAS 检查（需要时）
             ↓
返回 Translation Result
```

STE/CD 查询属于 **Configuration Table Walk**；使用其中的页表配置查找地址映射，才属于 **Translation Table Walk**。

## 对比总结

| 对比项 | STE | CD |
|---|---|---|
| 全称 | Stream Table Entry | Context Descriptor |
| 主要索引 | StreamID | SubstreamID/PASID |
| 配置粒度 | 设备数据流 | 进程或地址空间 |
| 核心职责 | 确定 stream 的总体翻译模式 | 提供 Stage-1 translation context |
| 典型内容 | Stage 选择、Stage-2 配置、CD Table 配置 | Stage-1 页表、ASID及相关控制信息 |
| 是否总要使用 | 每条受 SMMU 管理的 stream 先查 STE | 仅在相应 Stage-1/CD 配置需要时查询 |

## 一句话总结

> **STE 先根据 StreamID 决定“这个设备流怎么翻译”，CD 再根据 SubstreamID/PASID 决定“这个进程使用哪套 Stage-1 页表”；二者共同组成 translation context 的配置来源。**
