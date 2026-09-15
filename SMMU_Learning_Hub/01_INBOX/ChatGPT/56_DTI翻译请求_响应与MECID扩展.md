# DTI 翻译请求、响应与 MECID 扩展

记录日期：2026-09-08
记录端：本地

## 来源与整理范围

来源：[SMMU 对话](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)，2026-09-08 的相关问答。

本篇为 Inbox 来源记录（Capture），不是已查证晋升的正式知识。问题按原意凝练；回答保留原文，仅整理标题层级、排版及移除不可解析的引用标记。原回答中的“Slide 明确写”“TRM 明确说明”等为原答表述，本次未重新打开规范逐条复核。截图图像未随本篇保存，可回原会话查看；截图若来自培训视频，则为“培训视频截图，时间点未知”，原会话未提供视频时间戳。

## TRANS_REQ / RESP / FAULT 与乱序匹配

### 问题

DTI-TBU 翻译请求、成功响应和 Fault 各自携带什么？多个请求在途且乱序返回时如何匹配？

### 回答

这页讲的是 **DTI-TBU 最核心的一组消息：TBU 向 TCU 请求地址翻译，TCU 返回翻译结果或者 fault。**

> **一句话定义：`DTI_TBU_TRANS_REQ/RESP/FAULT` 就是 TBU 在本地 TLB miss 后，通过 DTI 把“这笔访问怎么翻”交给 TCU处理。**

###### 1. `DTI_TBU_TRANS_REQ`

TBU 发给 TCU，携带：

```text
IA
SID
SSID
SEC_SID
+ transaction attributes
```

其中：

- `IA`：Input Address，待翻译地址
- `SID`：找到对应 STE
- `SSID`：进一步找到 CD / Stage1 context
- `SEC_SID`：Stream 的 security state
- transaction attributes：供 permission / attribute checking 使用

可以理解成：

```text
TBU:
“SID=20、SSID=3 的访问，
地址 IA=0x1000，
请按这笔 transaction 的属性帮我完成 translation。”
```

---

###### 2. `DTI_TBU_TRANS_RESP`

TCU 成功完成 translation 后返回：

```text
OA
ASID / VMID
attributes
```

其中：

```text
OA = Output Address
```

也就是：

```text
IA
 ↓ SMMU translation
OA
```

例如：

```text
IA = IOVA 0x1000
 ↓
TCU
 ↓
OA = PA 0x80001000
```

另外还返回相关 translation attributes，TBU 后续可以：

```text
fill local TLB
+
继续原来的 transaction
```

所以：

> **REQ 给 TCU translation context；RESP 给 TBU最终 translation result。**

---

###### 3. `DTI_TBU_TRANS_FAULT`

如果 TCU 无法合法完成 translation：

```text
SID invalid
STE/CD invalid
Page-table fault
permission fault
...
```

就不是返回正常 `TRANS_RESP`，而是：

```text
DTI_TBU_TRANS_FAULT
```

Slide 特别提到它携带：

```text
fault type
+
这个fault能不能被cache
```

为什么 fault 也可能 cache？

假设某地址不断访问，而 translation context 明确导致相同 fault：

```text
第一次：
TBU → TCU → fault

第二次：
如果fault可cache
→ TBU可能不必再次向TCU请求
```

这样可以避免重复 fault lookup。

但具体哪些 fault 允许缓存，由 DTI/SMMU 架构规则决定，不能简单认为所有 fault 都能 cache。

---

##### 4. 什么时候发 TRANS_REQ？

最普通情况：

```text
Device transaction
       ↓
TBU
       ↓
MicroTLB miss
       ↓
Main TLB miss
       ↓
DTI_TBU_TRANS_REQ
       ↓
TCU
```

Slide 还写：

> `but can also be speculative`

意思是 TBU 也可以 **提前发 translation request**。

例如根据未来很可能会访问的地址：

```text
还没有真正需要这笔translation
        ↓
TBU speculative request
        ↓
提前拿到translation
        ↓
之后真正访问
        ↓
TLB hit
```

可以把它理解成：

> **translation prefetch。**

目的就是隐藏未来 TLB miss latency。

---

#### 5. 右边时序图为什么连续发三笔？

图中：

```text
TRANS_REQ(IA0, ..., TID0)
TRANS_REQ(IA1, ..., TID1)
TRANS_REQ(IA2, ..., TID2)
```

说明 DTI 支持：

> **multiple outstanding translation requests**

不需要：

```text
REQ0 → 等 RESP0
REQ1 → 等 RESP1
REQ2
```

而可以：

```text
REQ0
REQ1
REQ2
  ↓
都在途
```

这就是上一页 Translation Token 要限制的东西。

---

##### 6. Response 甚至可以乱序返回

图里 request：

```text
TID0
TID1
TID2
```

但是 response 顺序画成了类似：

```text
TID2
TID0
TID1
```

说明：

> **TCU 不一定按照 request 发出的顺序完成 translation。**

例如：

```text
IA2 → TLB/cache hit
IA0 → 需要较短walk
IA1 → page-table walk更慢
```

所以：

```text
RESP2
RESP0
RESP1
```

完全合理。

这就是为什么必须有 TID。

---

#### 7. `TID / TDEST` 在这里终于真正用起来了

Slide 最后一行：

> **Requests and responses have matching unique-in-flight TIDs/TDESTs**

请求：

```text
TBU → TCU

TRANS_REQ(..., TID=5)
```

TCU返回：

```text
TRANS_RESP(..., TDEST=5)
```

TBU看到：

```text
TDEST=5
```

就知道：

> “这是我之前 TID=5 那笔 translation request 的结果。”

所以：

```text
TID
→ Request 身份

TDEST
→ Response 要返回给哪个 request/context
```

而且是：

> **unique-in-flight**

即在当前 outstanding request 中，TID 需要能唯一识别对应事务。

这也和你刚才看的 BAS switch 完全串上了：

```text
TBU local TID
   ↓
BAS switch扩展成global TID
   ↓
TCU
   ↓
response TDEST
   ↓
BAS switch decode
   ↓
回到正确TBU + 正确request
```

---

##### 整页可以压缩成这一条链

```text
TBU local TLB miss
       ↓
TRANS_REQ
{IA, SID, SSID, attrs, TID}
       ↓
      TCU
       ↓
   translation
    /       \
 success    fault
   ↓          ↓
TRANS_RESP  TRANS_FAULT
{OA, attrs} {fault info}
   ↓
TDEST匹配原TID
   ↓
TBU继续transaction
```

###### 一句话记忆

> **`TRANS_REQ` 问“怎么翻”，`TRANS_RESP` 回“翻成什么”，`TRANS_FAULT` 回“为什么翻不了”；TID/TDEST 负责把乱序返回的结果和原请求重新对上。**

---

## TRANS_RESPEX：扩展响应与 MECID

### 问题

为什么引入 TRANS_RESPEX？REQEX=1 是否强制扩展响应，普通 RESP 时 MECID 如何解释，为什么影响 BAS 位宽？

### 回答

这页是在上一页 `DTI_TBU_TRANS_RESP` 的基础上，引入一个 **扩展版 Translation Response：`DTI_TBU_TRANS_RESPEX`**。

> **一句话定义：`TRANS_RESPEX` = 普通 `TRANS_RESP` + `MECID`，用于把 translation result 和 Memory Encryption Context 一起返回给 TBU。**

##### 为什么需要 RESPEX

普通 response 返回：

```text
OA
ASID / VMID
attributes
```

但开启 MEC 后，TBU 后续还需要知道：

```text
这笔 translated transaction
应该携带哪个 MECID？
```

所以扩展成：

```text
TRANS_RESPEX
=
TRANS_RESP
+
MECID
```

流程：

```text
TBU
 ↓ TRANS_REQ
TCU
 ↓ translation
OA + attrs + MECID
 ↓ TRANS_RESPEX
TBU
 ↓
最终向 system 发：
PA + attributes + MECID
```

---

##### `REQEX=1` 是什么意思

slide 说：

> TBU issues requests with `DTI_TBU_TRANS_REQ.REQEX = 1`

可以理解成 TBU 在 request 中告诉 TCU：

> **“如果可以，请给我 Extended Response，因为我需要额外的扩展信息。”**

也就是：

```text
REQEX = 0
→ 普通 response 能力

REQEX = 1
→ requester 支持/请求 RESPEX
```

---

##### 但 TCU 为什么“不是必须”返回 RESPEX？

slide 明确写：

> TCU can issue RESPEX ... but is not required to  
> it can still be `DTI_TBU_TRANS_RESP/FAULT`

也就是说：

```text
REQEX = 1
```

并不代表：

```text
TCU必须返回RESPEX
```

TCU仍可能返回：

```text
TRANS_RESP
```

或者：

```text
TRANS_FAULT
```

这是一种兼容设计。

---

##### 普通 RESP 返回时，MECID 怎么处理？

slide 最下面说得很关键：

> If `DTI_TBU_TRANS_RESP` is returned, equivalent to a RESPEX with all bits 0 in the MECID field.

也就是：

```text
TRANS_RESP
≈
TRANS_RESPEX
with MECID = 0
```

这样 TBU 处理逻辑可以统一理解：

```text
普通 RESP
→ MECID 看作 0

RESPEX
→ MECID 使用实际返回值
```

---

#### 为什么数据宽度从 160 bit 变成 192 bit？

普通：

```text
TRANS_RESP = 160 bits
```

加入 MECID 后：

```text
TRANS_RESPEX = 192 bits
```

所以你前面 BAS interconnect 那页才会说：

```text
upstream width
= 160 or 192 bits
depending on MECID width
```

现在就串起来了。

路径：

```text
TCU
 ↓ 192-bit RESPEX
BAS interconnect
 ↓
TBU
```

如果中间 BAS 链路比较窄：

```text
192-bit
 ↓
Sizer
 ↓
更窄link
```

所以 slide 才强调：

> **interconnect must be sized to account for this**

---

#### 和普通 RESP 的区别

| | `TRANS_RESP` | `TRANS_RESPEX` |
|---|---|---|
| OA | 有 | 有 |
| ASID/VMID | 有 | 有 |
| attributes | 有 | 有 |
| MECID | 无 | **有** |
| 宽度 | 160 bit | **192 bit** |
| DTI版本 | 普通 | DTI-TBUv3/v4 |

##### 一句话记忆

> **`TRANS_RESP` 回“地址怎么翻”；`TRANS_RESPEX` 在此基础上再告诉 TBU“这笔访问要带哪个 MECID”。**

而这页和我们前面 MEC 的讨论正好连起来：

```text
TCU决定 translation + MECID
        ↓
RESPEX
        ↓
TBU
        ↓
PA + MECID
        ↓
Memory system / encryption engine
```

## 一句话总结

TRANS_REQ 请求翻译，RESP/FAULT 返回结果；TID/TDEST 匹配在途事务，RESPEX 在翻译结果中补充 MECID。

## 整理注与待核实边界

原回答按截图介绍字段、160/192 bit 和 DTI-TBUv3/v4，具体产品与版本适用范围尚未在本次归档中核实。文中 speculative translation request 的提前请求含义，与 09 号记录所讨论的 TCU prefetch 触发条件是不同层面的讨论，复盘时不要直接混同。

## 关联记录

- [DTI协议概览与Token流控](./53_DTI协议概览与Token流控.md)
- [BAS互连组件与TID_TDEST路由](./55_BAS互连组件与TID_TDEST路由.md)
- [ACE5-Lite系统级Properties与翻译输出属性](./49_ACE5-Lite系统级Properties与翻译输出属性.md)
- [Speculative与Atomic请求不触发TCU_Prefetch](./09_Speculative与Atomic请求不触发TCU_Prefetch.md)

原问答标识：`b6963a52-a5e8-4688-a086-6d03b6b8a5fc`、`e37eb761-ba9b-4634-888c-33640db6164a`。
