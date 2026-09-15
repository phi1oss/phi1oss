# BAS 互连组件与 TID / TDEST 路由

记录日期：2026-09-08
记录端：本地

## 来源与整理范围

来源：[SMMU 对话](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)，2026-09-08 的相关问答。

本篇为 Inbox 来源记录（Capture），不是已查证晋升的正式知识。问题按原意凝练；回答保留原文，仅整理标题层级、排版及移除不可解析的引用标记。原回答中的“Slide 明确写”“TRM 明确说明”等为原答表述，本次未重新打开规范逐条复核。截图图像未随本篇保存，可回原会话查看；截图若来自培训视频，则为“培训视频截图，时间点未知”，原会话未提供视频时间戳。

## BAS 组件：汇聚、位宽、时序与跨域

### 问题

承载 DTI 的 BAS interconnect 包含哪些组件？Switch、Sizer、Register Slice 和 ADB-400 分别解决什么问题？

### 回答

这页是在讲：**DTI 本身定义消息语义，而 BAS interconnect 负责把这些 DTI 消息在多个 TBU / PCIe RP 和一个 TCU 之间真正“连起来、路由起来、跨域起来”。**

> **一句话定义：BAS interconnect 是 MMU-720AE 内部承载 DTI traffic 的可组合互连网络。**

###### 1. 为什么需要 BAS interconnect

一个系统可能有很多 requester：

```text
TBU0 ─┐
TBU1 ─┤
PCIe RP ─┤
TBU2 ─┘
        ↓
       TCU
```

它们都要和 TCU 通信，但 SoC 拓扑不一定一样，所以 Arm 没把内部连接结构固定死，而是：

> **Internal SMMU topology is user-defined**

也就是集成者可以按芯片需求组合：

```text
Switch
Sizer
Register Slice
ADB-400
```

来搭出自己的 DTI/BAS 网络。

---

##### 2. BAS switch：多路汇聚/路由

它解决：

> **多个 TBU / PCIe Root Port 怎么共享一个 TCU 接口。**

概念上：

```text
TBU0 ─┐
TBU1 ─┤
TBU2 ─┼→ BAS switch → TCU
PCIeRP─┘
```

Slide 明确写：

> connects multiple DTI subordinate interfaces to one DTI manager interface (TCU)

所以可以理解为：

> **BAS switch = DTI 消息交换/汇聚节点。**

它不是做 translation，而是负责把消息送到正确方向。

---

##### 3. BAS sizer：改变总线宽度

Slide：

> Upsizes/downsizes data width to reduce number of wires

它解决的是物理实现问题。

比如 TBU/TCU 端固定很宽：

```text
160-bit
或者 upstream 160/192-bit
```

但中间长距离布线如果一直保持这么宽：

```text
TBU ==================== TCU
        160/192 wires
```

routing、area、功耗压力会很大。

所以可以：

```text
160-bit
  ↓
Sizer
  ↓
更窄的 BAS link
  ↓
Sizer
  ↓
160-bit
```

本质是：

> **用更多 cycle 换更少的物理 wires。**

因此 Sizer 不改变 DTI message 的语义，只改变传输宽度。

---

##### 4. 为什么 upstream 可能是 160 或 192 bit

Slide 特别写：

```text
downstream:
固定 160 bit

upstream:
固定 160 或 192 bit
取决于 MECID width
```

这说明 MECID 会影响某个方向上的 message payload 宽度。

你前面刚学过 MECID，所以可以理解成：

```text
没有/较小 MECID
→ 160-bit

需要额外 MECID 信息
→ 192-bit
```

具体哪几个 bit 怎么排布，这页没有展开，不需要现在死记。

---

##### 5. BAS Register Slice：打断长 timing path

Slide：

> To improve timing by breaking long paths

这个最容易理解。

如果：

```text
TBU ─────────────很长组合路径────────────→ TCU
```

频率可能很难收敛。

插入 Register Slice：

```text
TBU
 ↓
logic
 ↓
Reg
 ↓
logic
 ↓
TCU
```

于是：

```text
一条很长路径
↓
拆成两条较短路径
```

所以：

> **Register Slice 主要解决 timing closure，不改变协议功能。**

---

##### 6. ADB-400：跨 clock / power domain

Slide：

> Asynchronous bridge between clock or power domains

如果：

```text
TBU clock = 800 MHz
TCU clock = 500 MHz
```

或者：

```text
TBU power domain
≠
TCU power domain
```

普通 BAS 组件不能直接解决这种异步/电源域问题。

于是：

```text
BAS
 ↓
ADB-400
 ↓
BAS
```

用来做：

```text
clock domain crossing
+
power domain crossing
```

这和你之前学 PPU / power domain 时的思路完全一致：

> **ADB-400 是边界桥接器，BAS switch/sizer/reg slice 本身不是 domain crossing component。**

---

#### 7. 右边图怎么读

图里其实展示了这几个组件可以自由组合：

```text
TBU0 ─┐
TBU1 ─┘
       ↓
   BAS switch
       ↓
     Sizer
       ↓
    ADB-400
       ↓
     Sizer
       ↓
   BAS switch
       ↓
      Reg
       ↓
      TCU
```

另一条：

```text
PCIe RP
   │
   └────────────→ BAS switch
```

另一 TBU：

```text
TBU
 ↓
Reg
 ↓
BAS switch
```

意思不是说所有系统必须长这样，而是：

> **这些 BAS component 像积木一样，根据 topology、timing、wire count、clock/power domain 自由拼装。**

---

##### 8. 把每个组件一句话记住

| 组件 | 解决的问题 |
|---|---|
| **BAS switch** | 多个 TBU/RP 怎么共享 TCU |
| **BAS sizer** | 总线太宽、布线太多怎么办 |
| **Register slice** | timing path 太长怎么办 |
| **ADB-400** | clock/power domain 不同怎么办 |

所以整页可以记成：

> **DTI 规定“消息是什么”，BAS interconnect 解决“这些消息在复杂 SoC 里怎么高效、可靠地送到 TCU”。**

---

## BAS Switch：请求 ID 扩展与响应路由

### 问题

BAS switch 如何通过独立 ID 区间消除多端口冲突，并利用 TDEST 将响应送回原端口？

### 回答

这页讲的是 **BAS switch 怎么把多个 subordinate 接口各自独立的 TID 空间，转换成 switch 后面统一且不冲突的全局 ID 空间**。

> **一句话定义：BAS switch 通过给每个 S 口分配一段独立的 ID range，把各端口重复的本地 TID 映射成 M 口唯一的 TID；返回方向再根据 TDEST 把消息送回正确的 S 口。**

##### 1. 为什么需要“unique namespace”

假设有 4 个 TBU：

```text
S0: TID = 0
S1: TID = 0
S2: TID = 0
S3: TID = 0
```

每个 TBU 自己内部看都没有问题，因为它们属于不同接口。

但经过 BAS switch 汇聚到一个 TCU：

```text
S0 ─┐
S1 ─┤
S2 ─┼→ BAS Switch → TCU
S3 ─┘
```

如果全部仍然叫：

```text
TID = 0
```

TCU 返回 response 时就无法知道：

> 这个 `TID=0` 到底属于哪个 TBU？

所以 switch 必须把它们转换成统一的全局 ID。

---

#### 2. `DECMIN_Sn / DECMAX_Sn` 是干什么的

它们给每个 S 接口分配一段 **全局 ID range**。

slide 的例子：

| 接口 | 本地 TID | 全局范围 |
|---|---|---|
| S0 | 0~5 | 0~5 |
| S1 | 0~3 | 6~9 |
| S2 | 0~3 | 10~13 |
| S3 | 0~7 | 14~21 |

例如 S1：

```text
DECMIN_S1 = 6
DECMAX_S1 = 9
```

所以：

```text
S1 local TID 0 → global TID 6
S1 local TID 1 → global TID 7
S1 local TID 2 → global TID 8
S1 local TID 3 → global TID 9
```

可以近似理解成：

```text
Global_TID = Local_TID + DECMIN_Sn
```

---

#### 3. `Extends TIDs from DN_Sn to DN_M`

这就是请求方向：

```text
TBU / RP
   ↓ DN_Sn
BAS switch
   ↓ DN_M
TCU
```

例如：

```text
S2:
local TID = 2
DECMIN = 10

        ↓ BAS switch

M side TID = 12
```

于是 TCU 看到的：

```text
0~5   → 一定来自 S0
6~9   → 一定来自 S1
10~13 → 一定来自 S2
14~21 → 一定来自 S3
```

所以：

> **TID extension 的目的就是避免不同 TBU 的 transaction ID 冲突。**

---

#### 4. `Reduces TDESTs from UP_M to UP_Sn`

返回方向刚好反过来：

```text
TCU
 ↓ UP_M
BAS Switch
 ↓ UP_Sn
TBU
```

TCU 返回的 message 里带一个全局 `TDEST`。

例如：

```text
TDEST = 12
```

Switch 看配置：

```text
10 <= 12 <= 13
```

所以知道：

```text
这个response应该送到 S2
```

然后在 S2 一侧恢复本地编号：

```text
12 - DECMIN_S2(10)
= 2
```

于是：

```text
Global TDEST = 12
       ↓
BAS Switch
       ↓
S2
       ↓
Local TDEST = 2
```

所以可以记：

```text
请求方向：
Local TID → Global TID

返回方向：
Global TDEST → 找到 S口 + Local TDEST
```

---

#### 5. 为什么各个 range 不能重叠

Slide 明确要求：

> Each `{DECMIN_Sn, DECMAX_Sn}` range must be unique

如果：

```text
S0 = 0~5
S1 = 4~7
```

那么收到：

```text
TDEST = 4
```

Switch 就不知道该：

```text
→ S0
```

还是：

```text
→ S1
```

所以 range 必须互斥。

---

#### 6. `ID_WIDTH` 怎么确定

这个例子最大 global ID 是：

```text
21
```

需要能够表示：

```text
0 ~ 21
```

4 bit：

```text
0~15   不够
```

5 bit：

```text
0~31   足够
```

因此：

> **这个例子的 `ID_WIDTH = 5`。**

---

#### 一句话记忆

> **每个 TBU 可以都有自己的 `TID=0`；BAS switch 通过给每个接口分配不同的 ID range，把它们转换成全局唯一 TID，TCU 返回时再利用 TDEST 的 range 找回原来的 TBU。**

可以把它类比成：

> **TID 是“房间号”，BAS switch 给它加上“楼栋信息”，否则不同楼里的 101 室会冲突。**

---

## 追问：TID 与 DECMIN_Sn / DECMAX_Sn

### 问题

这里的 TID 是什么？DECMIN_Sn 和 DECMAX_Sn 分别表示什么？

### 回答

可以先把这三个东西理解成 **BAS switch 用来解决“多个端口的 ID 冲突和返回路由”** 的机制。

> **TID = BAS/AXI5-Stream 消息携带的 ID，用来标识消息所在的 ID namespace。**  
> **DECMIN_Sn / DECMAX_Sn = BAS switch 给第 n 个 subordinate 端口分配的全局 ID 区间上下界。**

#### 1. TID 是什么

因为 BAS 承载在 AXI5-Stream 上，所以消息会带 `TID`。

例如两个 TBU 都可能合法产生：

```text
TBU0: TID = 0
TBU1: TID = 0
```

在各自接口内部没问题，因为每个 subordinate interface 都有自己的独立 namespace。

但汇聚到同一个 BAS switch 后：

```text
TBU0 ─┐
TBU1 ─┼→ BAS switch → TCU
TBU2 ─┘
```

如果仍然都是 `TID=0`，就会发生冲突。

因此 switch 要把本地 TID 扩展成 **全局唯一 TID**。

---

#### 2. DECMIN / DECMAX 是什么

看 slide：

```text
S0 local TID: 0~5
DECMIN_S0 = 0
DECMAX_S0 = 5

S1 local TID: 0~3
DECMIN_S1 = 6
DECMAX_S1 = 9

S2 local TID: 0~3
DECMIN_S2 = 10
DECMAX_S2 = 13
```

意思就是给不同 S 口分配不重叠的全局 ID 空间：

```text
Global ID space

0 ───── 5   → S0
6 ───── 9   → S1
10 ──── 13  → S2
14 ──── 21  → S3
```

所以对于 S1：

```text
local TID 0
    ↓ BAS switch
global TID 6

local TID 1
    ↓
global TID 7
```

概念上可以理解成：

```text
global_TID ≈ DECMIN_Sn + local_TID
```

这张 slide 表现出的就是这种映射关系。

---

#### 3. DECMIN / DECMAX 为什么叫 DEC？

可以把 `DEC` 理解成 **decode range**。

返回方向 TCU 发来：

```text
TDEST = 12
```

BAS switch 看：

```text
10 <= 12 <= 13
```

于是知道：

```text
这是 S2 的消息
```

再把它恢复成 S2 本地 namespace：

```text
12 → local TDEST 2
```

因此：

```text
DECMIN_Sn
→ 这个端口全局ID范围的起点

DECMAX_Sn
→ 这个端口全局ID范围的终点
```

---

#### 4. TID 和 TDEST 的关系

这页可以这样理解：

```text
请求方向：

TBU
local TID
   ↓
BAS switch
   ↓ extend
global TID
   ↓
TCU
```

返回方向：

```text
TCU
global TDEST
   ↓
BAS switch
   ↓ decode DECMIN/DECMAX
   ↓
找到对应 S口
   ↓ reduce
local TDEST
```

所以：

> **TID 帮助建立全局唯一身份，TDEST 帮助返回消息找到正确的 subordinate interface。**

#### 一句话记忆

> **TID 是“本地编号”，BAS switch 用 DECMIN/DECMAX 给不同端口划分不同的“全局号段”；返回时再根据这个号段把消息送回正确的 TBU。**

## 一句话总结

BAS 组件负责承载和传送 DTI 消息；Switch 通过 ID 区间映射区分来源，再用 TDEST 完成返回路由。

## 整理注与待核实边界

原回答的 BAS 接口 manager/subordinate 表述与连接管理篇中的 DTI 角色称呼存在易混淆之处；此处保留原文，复盘时须按各自协议层和端口视角核实，不能直接互相替换。“全局 ID”沿用图示汇聚范围的说法；加减 DECMIN 的公式是原回答对该示例的解释，非本次独立核实的普遍编码规则。

## 关联记录

- [DTI协议概览与Token流控](./53_DTI协议概览与Token流控.md)
- [DTI连接管理与Token生命周期](./54_DTI连接管理与Token生命周期.md)
- [DTI翻译请求_响应与MECID扩展](./56_DTI翻译请求_响应与MECID扩展.md)

原问答标识：`b3604022-be99-4385-9a05-6f36f635a445`、`e4c3b918-d77d-4306-95ec-adf6a35aa0e7`、`5ea96aa7-b93b-4b1a-a3ad-da838dc245a0`。
