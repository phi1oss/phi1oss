---
title: "VC Static RDC：SETUP_RESET_UNDECL 的含义与处理流程"
received_at: "2026-09-22 17:08 +0800"
source: "ChatGPT RDC 对话"
status: "inbox"
topics:
  - RDC
  - VC Static
  - VC SpyGlass
  - reset constraint
  - SETUP_RESET_UNDECL
---

# VC Static RDC：SETUP_RESET_UNDECL 的含义与处理流程

## 结论

当前大量 `SETUP_RESET_UNDECL` 应视为 **RDC setup 尚未完成**，不是普通的 RDC 设计 violation。

工具已经从 RTL 中发现了连接到寄存器异步 set/reset pin 的信号，但这些 reset 没有被正式声明、传播或覆盖。相关寄存器可能无法正常进入 RDC 分析，因此即使后续 RDC violation 很少甚至为 0，也不能认为 RDC clean。

```text
RTL
 │
 ├── clock SDC                 ✓
 ├── reset constraint          ✗
 │
 ▼
VC Static 找到 FF 的 async reset/set pin
 │
 ▼
对应 reset 尚未正式声明或传播
 │
 ▼
SETUP_RESET_UNDECL
 │
 ▼
RDC setup coverage 不完整
```

## 一、先自动推导 reset，生成检查起点

在已经完成 elaboration 的 VC Static 环境中，可先尝试：

```tcl
infer_setup -type reset -sync_resets false -full

write_inferred_setup \
    -file tool_inferred_resets.sdc \
    -type reset
```

随后检查 `tool_inferred_resets.sdc`。工具可能生成类似内容：

```tcl
create_reset {por_n} \
    -async \
    -type reset \
    -sense low

create_reset {cpu_rst_n} \
    -async \
    -type reset \
    -sense low
```

不同版本的命令选项可能有差异，应在当前安装版本中用以下命令核对：

```tcl
help infer_setup
help write_inferred_setup
help create_reset
```

## 二、不要把自动生成结果直接当作 signoff 约束

自动推导文件只是 reset inventory 的起点，需要人工 review：

| 检查对象 | 需要确认的问题 |
|---|---|
| `por_n` | 是否为真正的 primary reset |
| `cpu_rst_n`、`peri_rst_n` | 是 primary reset 还是内部 generated reset |
| `sw_rst_n`、`wd_rst_n` | 是否可独立触发，工作模式下是否有效 |
| `scan_rst_n`、`mbist_rst_n` | 是否仅在 DFT/test mode 使用 |
| 内部信号 | 是否被工具误识别为 reset |
| reset polarity | active high/low 是否正确 |
| reset fanout | 关联的 sequential cell 数量是否符合预期 |

目标流程应是：

```text
infer_setup
    ↓
发现所有可能的 reset
    ↓
人工对照 reset architecture
    ↓
形成正式 rdc_reset.sdc
```

## 三、建立正式 reset constraint

示意：

```tcl
# Primary POR
create_reset {por_n} \
    -async \
    -type reset \
    -sense low

# Software reset
create_reset {sw_rst_n} \
    -async \
    -type reset \
    -sense low

# Watchdog reset
create_reset {wd_rst_n} \
    -async \
    -type reset \
    -sense low
```

如果设计结构为：

```text
por_n
  |
reset controller
  |
  +---- cpu_rst_n
  |
  +---- peri_rst_n
```

不能简单地把三者都声明为互不相关的异步 reset。还需要根据真实架构描述：

- primary 与 generated reset 的关系；
- reset 之间的同步/异步关系；
- assertion/deassertion 时序；
- reset sequencing；
- mutual exclusion；
- reset qualifier；
- reset 与 clock/power mode 的关系。

## 四、逐条追查残留的 SETUP_RESET_UNDECL

假设报告给出：

```text
FF        : u_cpu/u_core/u_reg/q_reg
reset pin : RN
reset net : cpu_rst_n
```

沿 reset cone 向上追踪：

```text
q_reg/RN
   ↑
cpu_rst_n
   ↑
reset_sync
   ↑
por_n
```

依次确认：

1. `por_n` 是否已被 `create_reset`；
2. `cpu_rst_n` 是否属于 generated/local reset；
3. 工具是否成功把 reset 属性从根 reset 传播到 `cpu_rst_n`；
4. 中间是否经过 black box、reset controller、mux 或工具不支持的组合结构；
5. 是否存在 undeclared net、错误 polarity 或 mode constraint 缺失。

因此，报错的根因不一定只是漏写一个 `create_reset`，也可能是 reset propagation 在中间被截断。不要看到每个内部 reset net 就机械地补一条独立 `create_reset`。

## 五、重新运行时的阶段目标

第一阶段目标不是 RDC violation 清零，而是：

- reset inventory 与设计架构一致；
- reset polarity 正确；
- 每个 reset 覆盖的 sequential cells 数量合理；
- `SETUP_RESET_UNDECL` 接近 0；
- 少量残留项都有明确、可审查的解释。

推荐顺序：

```text
SETUP_RESET_UNDECL 很多
        ↓
infer_setup -type reset
        ↓
write_inferred_setup
        ↓
人工 review reset
        ↓
建立正式 rdc_reset.sdc
        ↓
与 clock SDC 一起重新运行
        ↓
清理 SETUP_RESET_UNDECL
        ↓
检查 reset relationship / sequencing
        ↓
分析真正 RDC violations
        ↓
RTL fix / constraint / justified waiver
```

示意读取：

```tcl
read_sdc clock.sdc
read_sdc rdc_reset.sdc

# 实际分析命令以项目 flow 和当前工具版本为准
check_rdc
```

## 六、关键判断

不要用“RDC violation 数量”判断当前 flow 是否可信，先看 setup coverage：

> `create_reset` 不是 RDC 报错的唯一前提；reset 被 RDC engine 正确识别、传播并建立 reset domain，才是有效分析的前提。

在 `SETUP_RESET_UNDECL` 尚未处理完成时，当前结果不能用于 RDC signoff。

## 参考资料

- [Synopsys：RDC Signoff Steps](https://www.synopsys.com/zh-cn/blogs/chip-design/rdc-signoff-steps.html)
- [Synopsys：VC SpyGlass RDC](https://www.synopsys.com/verification/static-and-formal-verification/vc-spyglass/vc-spyglass-rdc.html)
- 具体命令语法与 rule 行为以项目所用版本的 VC Static / VC SpyGlass RDC User Guide 和内置 `help` 为准。
