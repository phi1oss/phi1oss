# DTI-ATS Page Request 与 PRI 恢复闭环

记录日期：2026-09-08
记录端：本地

## 来源与整理范围

来源：[SMMU 对话](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)，2026-09-08 的相关问答。

本篇为 Inbox 来源记录（Capture），不是已查证晋升的正式知识。问题按原意凝练；回答保留原文，仅整理标题层级、排版及移除不可解析的引用标记。原回答中的“Slide 明确写”“TRM 明确说明”等为原答表述，本次未重新打开规范逐条复核。截图图像未随本篇保存，可回原会话查看；截图若来自培训视频，则为“培训视频截图，时间点未知”，原会话未提供视频时间戳。

## PAGE_REQ / ACK / RESP / RESPACK

### 问题

ATS 翻译失败后，如何经 DTI-ATS Page Request、PRIQ 和软件处理恢复？PAGE_ACK、PAGE_RESP 与 PAGE_RESPACK 有何区别？

### 回答

这页讲的是 **PCIe PRI（Page Request Interface）在 DTI-ATS 里的完整请求/恢复流程**。

> **一句话定义：当 PCIe Endpoint 的 ATS translation 失败后，它可以通过 PRI 请求软件把缺失/不可用的页面准备好；TCU 把请求放进 PRIQ，软件修复后再通过 TCU 把结果返回给 Endpoint。**

##### 1. 为什么需要 Page Request

正常 ATS：

```text
PCIe Endpoint
   ↓ ATS_TRANS_REQ
TCU
   ↓
translation success
   ↓ ATS_TRANS_RESP
Endpoint
```

但如果 translation 失败，例如相关 mapping 尚不存在：

```text
ATS_TRANS_REQ
    ↓
TCU
    ↓
ATS_TRANS_FAULT
```

如果直接失败，这次 Device 工作只能结束。

PRI 提供了另一种机制：

> **“现在翻不了，你先告诉软件，让软件把 page 准备好，然后我再继续。”**

---

##### 2. `DTI_ATS_PAGE_REQ`

Endpoint 通过 PCIe RC 向 TCU 发送 Page Request：

```text
Endpoint
   ↓ PCIe PRI
PCIe RC
   ↓ DTI_ATS_PAGE_REQ
TCU
```

Slide 中它携带类似：

```text
ADDR
SID
SSID
PRG
```

表示：

> 哪个 Stream/Substream 对哪个地址产生了 Page Request。

TCU 随后把这个请求提供给软件。

在 SMMUv3 中就是：

```text
PAGE_REQ
   ↓
TCU
   ↓
PRIQ
   ↓
Software
```

---

##### 3. `PAGE_ACK` 不是“页面已经准备好了”

这点和前面 `INV_ACK` 很像。

Slide 明确说：

> `DTI_ATS_PAGE_ACK` — Simply an ack

所以：

```text
PAGE_REQ
   ↓
TCU

PAGE_ACK
   ↑
```

主要表示：

> **Page Request 已经被接收。**

不是：

> “translation fault 已经修好了。”

真正修复还得等软件。

---

##### 4. 软件做什么

软件从 PRIQ 读到请求：

```text
PRIQ record
ADDR / SID / SSID / ...
      ↓
Software
```

然后软件可能：

```text
分配物理页
修改page table
设置permission
必要的TLBI/同步
```

总之让原来失败的 translation 变得可用。

完成后，软件向 TCU 发 PRI response command。

---

##### 5. `DTI_ATS_PAGE_RESP`

然后 TCU：

```text
TCU
 ↓ DTI_ATS_PAGE_RESP
PCIe RC
 ↓
Endpoint
```

告诉 Endpoint：

> **这个 Page Request 的处理结果出来了。**

Slide 里写：

> `Issued by TCU when SW has made page available`

所以正常成功场景可以理解成：

```text
“你刚才缺的page现在准备好了”
```

Endpoint 后面可以重新发 ATS Translation Request：

```text
PAGE_RESP
   ↓
ATS_TRANS_REQ again
   ↓
TCU
   ↓
ATS_TRANS_RESP
```

---

##### 6. `PAGE_RESPACK`

同理：

```text
TCU → PAGE_RESP → PCIe RC
TCU ← PAGE_RESPACK
```

`PAGE_RESPACK` 也只是 acknowledgement：

> **确认 Page Response 被接收。**

---

#### 整个流程串起来

```text
PCIe Endpoint
     │
     │ ATS Translation Request
     ▼
PCIe RC
     │ ATS_TRANS_REQ
     ▼
    TCU
     │
     │ translation fails
     ▼
ATS_TRANS_FAULT
     │
     ▼
Endpoint
     │
     │ PCIe PRI Page Request
     ▼
PCIe RC
     │
     │ DTI_ATS_PAGE_REQ
     ▼
    TCU
     │
     ├─ PAGE_ACK
     │
     └─ 写入 PRIQ
            ↓
         Software
            ↓
     修复 mapping/page
            ↓
         TCU
            │
            │ PAGE_RESP
            ▼
        PCIe RC
            ↓
         Endpoint
            │
      再次 ATS_TRANS_REQ
            ↓
           TCU
            ↓
       ATS_TRANS_RESP
```

##### 这页最值得记的三个概念

> **ATS_TRANS_FAULT：现在翻不了。**

> **PAGE_REQ：请软件帮我把这个 page 准备好。**

> **PAGE_RESP：软件已经处理完了，你可以继续了。**

另外：

> **PAGE_ACK / PAGE_RESPACK 都只是协议握手确认，不代表 page repair 本身完成。**

这页本质上体现的是：**Device 的 translation fault 不一定立即终止，可以通过 PRI → PRIQ → Software 形成一个可恢复的缺页处理闭环。**

## 一句话总结

Page Request 将页面准备请求交给软件；PAGE_ACK/RESPACK 是接收确认，后续是否重试要看 Page Response 的处理结果。

## 整理注与待核实边界

原回答以软件成功准备页面后的恢复路径为主；“你可以继续了”应在该成功场景下理解。失败响应、PRG 细节及重试条件未在原回答中展开；示意流程中的 ATS_TRANS_* 名称不作为本次核实过的精确协议编码。

## 关联记录

- [PRI缺页恢复与无PRI退化路径](./37_PRI缺页恢复与无PRI退化路径.md)
- [PCIe_ATS与ATC工作流程](./35_PCIe_ATS与ATC工作流程.md)
- [DTI失效与同步_INV_ACK和SYNC_ACK](./57_DTI失效与同步_INV_ACK和SYNC_ACK.md)

原问答标识：`ea6a0cad-5b15-4cb4-8e6d-3ff5a29dd148`。
