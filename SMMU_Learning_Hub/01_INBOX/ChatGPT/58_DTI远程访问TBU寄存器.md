# DTI 远程访问 TBU 寄存器

记录日期：2026-09-08
记录端：本地

## 来源与整理范围

来源：[SMMU 对话](https://chatgpt.com/c/6a8448ed-6068-83ee-93ab-d6bdfe2461e8)，2026-09-08 的相关问答。

本篇为 Inbox 来源记录（Capture），不是已查证晋升的正式知识。问题按原意凝练；回答保留原文，仅整理标题层级、排版及移除不可解析的引用标记。原回答中的“Slide 明确写”“TRM 明确说明”等为原答表述，本次未重新打开规范逐条复核。截图图像未随本篇保存，可回原会话查看；截图若来自培训视频，则为“培训视频截图，时间点未知”，原会话未提供视频时间戳。

## DTI_TBU_REG_* 读写路径

### 问题

软件如何通过 TCU 的统一编程接口访问远端 TBU 寄存器？为什么寄存器访问请求由 TCU 发起？

### 回答

这页讲的是：**TCU 可以通过 DTI 去访问某个 TBU 内部的寄存器，相当于把 TBU 的寄存器访问“代理”到 TCU 的统一 programming interface 上。**

> **一句话定义：`DTI_TBU_REG_*` 是 TCU 与 TBU 之间的寄存器远程访问协议，用于读写 TBU-specific registers。**

##### 1. 为什么需要这种机制

TBU 通常分散在 SoC 各处：

```text
TBU0        TBU1        TBU2
 |           |           |
 +-----------DTI---------+
             |
            TCU
             |
      Programming Interface
             |
            CPU
```

如果软件要访问每个 TBU 的内部寄存器，没必要给每个 TBU 都单独做一套 CPU programming bus。

于是可以：

```text
CPU
 ↓
TCU programming interface
 ↓
TCU发现：这是TBU-specific register
 ↓
通过DTI转发到对应TBU
```

所以 TCU 相当于一个 **TBU register access proxy**。

---

#### 2. Read 流程

Slide 右边：

```text
TCU
 │
 │ DTI_TBU_REG_READ(ADDR, NS)
 ▼
TBU
 │
 │ 读取本地register
 │
 │ DTI_TBU_REG_RDATA(DATA)
 ▼
TCU
```

也就是：

> TCU 发 `REG_READ`，TBU 返回 `REG_RDATA`。

例如软件想读 TBU 的 RAS status：

```text
CPU
 ↓
TCU programming interface
 ↓
REG_READ
 ↓ DTI
TBU RAS register
 ↓
RDATA
 ↓
CPU
```

---

#### 3. Write 流程

写寄存器：

```text
TCU
 │
 │ DTI_TBU_REG_WRITE(ADDR, NS, DATA)
 ▼
TBU
 │
 │ 完成写入
 │
 │ DTI_TBU_REG_WACK
 ▼
TCU
```

所以：

```text
REG_WRITE
→ 发地址 + 数据

WACK
→ 告诉TCU“写操作完成”
```

注意 `WACK` 不是返回数据，只是 completion acknowledgement。

---

#### 4. 为什么图里看起来是 TCU 主动发 Request？

这点确实容易迷惑，因为顶部又写：

```text
DTI manager      = TBU
DTI subordinate  = TCU
```

但 **“DTI manager/subordinate”描述的是 DTI connection 的角色，并不意味着所有 message 永远只能由 manager 发起。**

在 Register Access 这个特定功能里：

```text
TCU → TBU
```

发送 `REG_READ/REG_WRITE`，因为真正要被访问的 register 就在 TBU 内。

所以不要简单理解：

> manager 永远是 request source。

具体要看是哪一类 DTI message。

---

#### 5. 哪些寄存器会这样访问？

Slide 给了三个例子：

```text
TBU u-Arch registers
TBU RAS registers
TBU MPAM registers
```

也就是偏 TBU 本地实现的东西，例如：

- micro-architecture 状态/配置
- RAS error record/status
- MPAM 相关配置/监控

而不是普通：

```text
STE
CD
Page Table
```

这些 memory-based translation structures。

---

#### 6. 为什么只适用于 TBU？

Slide 明确写：

> **Applies to TBU only**

因为这个消息本身就是：

```text
DTI_TBU_REG_*
```

设计目的就是让中央 TCU programming interface 能访问分布式 TBU 的本地寄存器。

可以理解成：

```text
Software只面对TCU
       ↓
TCU作为统一管理入口
       ↓
DTI
       ↓
访问远端TBU register
```

---

##### 一句话记忆

> **DTI Register Access = “软件从 TCU 这个统一入口，远程读写分布在 SoC 各处的 TBU 本地寄存器”。**

Read：

```text
TCU → REG_READ → TBU
TCU ← RDATA    ← TBU
```

Write：

```text
TCU → REG_WRITE → TBU
TCU ← WACK      ← TBU
```

这页真正体现的是 **分布式 TBU + 集中式软件管理** 的设计思路。

## 一句话总结

DTI Register Access 将分布式 TBU 的本地寄存器访问汇聚到 TCU 编程入口；请求方向取决于具体消息功能。

## 整理注与待核实边界

此处记录培训截图中的 TBU-specific register 代理访问，不把它扩展为 STE/CD/页表访问。地址映射、权限、错误路径和 WACK 的精确完成约束待查证。

## 关联记录

- [DTI连接管理与Token生命周期](./54_DTI连接管理与Token生命周期.md)
- [SMMUv3寄存器空间与Memory-based配置](./26_SMMUv3寄存器空间与Memory-based配置.md)

原问答标识：`ad3149fc-e699-497e-baa5-af0aa28a6459`。
