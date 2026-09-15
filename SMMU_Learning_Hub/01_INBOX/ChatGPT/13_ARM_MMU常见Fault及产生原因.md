# ARM MMU 常见 Fault 及产生原因

## 问题

Arm MMU 常见的同步 fault 有哪些？它们分别为什么产生？

## 核心结论

**MMU fault 是地址翻译过程中，因为地址不合法、映射不存在、访问权限不允许、访问状态不满足，或者 TLB 中出现矛盾 translation 而触发的同步异常。**

常见五类包括：

```text
Permission Fault
Translation Fault
Address Size Fault
Access Flag Fault
TLB Conflict Abort
```

## 1. Permission Fault：映射存在，但访问没有权限

地址翻译本身已经成功，VA 能够映射到 PA，但当前访问类型不满足页表中的权限。

例如，PTE 将页面配置成 Read Only，而 CPU 执行写操作：

```text
VA
 ↓
找到有效 PTE
 ↓
得到 PA
 ↓
检查 AP 权限
 ↓
当前是 Write，但只允许 Read
 ↓
Permission Fault
```

典型原因包括：

- 写入只读页面。
- EL0 访问只允许 EL1 使用的页面。
- 执行 UXN/PXN 页面。
- Stage-2 权限不允许当前访问。

> **一句话：地址能翻译，但你没有权限这样访问。**

## 2. Translation Fault：找不到有效地址映射

Translation Fault 表示 MMU 在页表遍历过程中没有找到有效的最终映射。

```text
L0 Descriptor Valid
        ↓
L1 Descriptor Valid
        ↓
L2 Descriptor Invalid
        ↓
无法继续 Table Walk
        ↓
Translation Fault
```

最常见原因是目标 VA 根本没有建立映射，或某一级 descriptor 使用 invalid encoding。

Translation Fault 不应简单等同于操作系统中的所有 Page Fault。Page Fault 是更高层的软件概念，可以由不同 MMU fault 情况触发。

> **一句话：页表中没有通向 PA 的有效路径。**

## 3. Address Size Fault：地址超出允许的宽度

Address Size Fault 表示输入地址或页表给出的输出地址，超出了当前 translation regime 或硬件支持的地址宽度。

可能包括：

```text
输入 VA 的高位不符合当前 VA Size 规则

或

Descriptor 给出的 Output Address
超出了支持的 PA/OA Size
```

例如系统只支持 48-bit PA，但 descriptor 中超出该范围的高位被非法设置。这不是“没有 PTE”，而是地址数值本身无法由当前配置合法表示。

> **Translation Fault 看有没有映射；Address Size Fault 看地址能不能表示。**

## 4. Access Flag Fault：尚未标记为已访问

当 PTE 有效、权限也正确，但 `AF=0`，并且没有启用硬件 Access Flag 自动更新时，第一次访问会产生 Access Flag Fault。

```text
访问有效 PTE
      ↓
    AF = 0
      ↓
Access Flag Fault
      ↓
软件异常处理程序将 AF 置 1
      ↓
执行必要的 TLB Maintenance
      ↓
    Retry
```

如果启用了硬件 AF update，并且实现支持相应的 HTTU 能力，硬件可以自动把 AF 从 0 更新为 1，而不必产生 fault。

它通常不是表示访问真正非法，而是可以被软件用来跟踪页面是否被访问过。

> **一句话：映射和权限都有效，但页面尚未标记为“访问过”。**

## 5. TLB Conflict Abort：存在冲突的有效 Translation

TLB Conflict Abort 表示，对同一个有效 lookup 条件，TLB 中出现了多个彼此不一致、无法唯一选择的 translation result。

```text
TLB Entry A：VA 0x4000 → PA 0x8000

TLB Entry B：VA 0x4000 → PA 0x9000
```

两项对当前 VA、ASID 等 lookup 条件都有效，但结果不同。硬件无法安全地任意选择其中一个，因此产生 TLB Conflict Abort。

典型根源是软件修改了 translation table，却没有正确执行 TLB maintenance 或 Break-Before-Make，导致旧 translation 与新 translation 同时可见。

```text
Break
  ↓
TLBI
  ↓
Synchronization
  ↓
Make
```

> **一句话：TLB 找到了不止一个结果，而且这些结果互相冲突。**

## 五类 Fault 在 Translation Flow 中的位置

```text
CPU 访问 VA
    ↓
检查地址宽度是否合法
    └─ 不合法 → Address Size Fault
    ↓
TLB Lookup
    ├─ 多个冲突结果 → TLB Conflict Abort
    └─ Miss
         ↓
      Table Walk
         ↓
检查 Descriptor 是否有效
    └─ 无效 → Translation Fault
    ↓
检查 Access Flag
    └─ AF=0 且不能自动更新
          → Access Flag Fault
    ↓
检查访问权限
    └─ 不允许 → Permission Fault
    ↓
得到 PA
    ↓
Memory Access
```

## 关键对比

```text
Translation Fault
“没有路。”
页表中找不到有效 Mapping

Permission Fault
“有路，但你不能走。”
Mapping 存在，但当前读、写或执行权限不满足

Address Size Fault
“地址本身超出规则。”
地址宽度不符合当前 Translation Regime

Access Flag Fault
“有路且有权限，但尚未标记访问状态。”

TLB Conflict Abort
“找到多条路，而且结果互相冲突。”
```

## Fault 发生后如何定位原因

MMU fault 会触发同步异常。软件可以结合 `ESR_ELx` 与 `FAR_ELx` 分析：

```text
ESR_ELx
→ 判断异常类型、Fault 类型和 Translation Level

FAR_ELx
→ 记录相关 Fault Address
```

## 面试回答

> **MMU fault 是地址翻译或权限检查过程中产生的同步异常，常见包括 Translation Fault、Permission Fault、Address Size Fault、Access Flag Fault 和 TLB Conflict Abort。Translation Fault 表示页表中找不到有效 mapping；Permission Fault 表示 mapping 存在，但当前访问权限不满足；Address Size Fault 表示输入或输出地址超出当前 translation regime 支持的范围；Access Flag Fault 表示访问 AF=0 的有效 mapping，若支持硬件 AF 更新则可以避免；TLB Conflict Abort 表示 TLB 中存在相互冲突的有效 translation，通常与错误的页表更新或 TLB maintenance 有关。**

## 一句话记忆

> **Address Size：地址太大；Translation：找不到映射；Access Flag：还没标记“访问过”；Permission：找到但没权限；TLB Conflict：找到多个且结果冲突。**
