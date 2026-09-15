# Configuration Table Walk 与 GPT Walk

TCU 收到 translation request 后，除了执行真正的地址翻译，还需要确定翻译配置，并在需要时检查目标物理地址的安全归属。Configuration Table Walk 和 GPT Walk 分别承担这两类工作。

## 一、Configuration Table Walk

### 问题

如何理解下面这句话？

> Configuration table walks, that return configuration information for the translation context.

### 核心结论

**Configuration Table Walk 不是在翻译地址，而是在根据设备身份找到“这次地址翻译应该采用什么配置”。它返回的是 translation context，后续的 Translation Table Walk 才使用这些信息完成地址翻译。**

### 解析

1. **Configuration information 是地址翻译的前提。**

   设备向 SMMU 发起访问时，通常会携带 `StreamID`，使用进程地址空间时还可能携带 `SubstreamID/PASID`。TCU 必须先确定这是哪个设备或进程的请求，以及应该使用哪套地址翻译配置。

2. **Configuration Table Walk 主要查找两类配置结构。**

   ```text
   StreamID
      ↓
   Stream Table Walk
      ↓
   Stream Table Entry（STE）
      ↓
   是否启用 Stage 1 / Stage 2？
   使用哪个 VMID？
   Stage-2 页表在哪里？
   是否需要继续查询 Context Descriptor？
      ↓
   SubstreamID
      ↓
   Context Descriptor Table Walk
      ↓
   Context Descriptor（CD）
      ↓
   Stage-1 页表在哪里？
   使用哪个 ASID？
   地址空间和翻译属性是什么？
   ```

   `STE` 描述设备数据流的总体翻译方式；需要 Stage 1 翻译时，`CD` 进一步描述具体进程或地址空间的翻译配置。

3. **它返回 translation context，而不是最终 PA。**

   Configuration Table Walk 得到的信息可以包括：

   ```text
   Stage 1：启用
   Stage 2：启用
   ASID：A
   VMID：B
   Stage-1 页表基地址：X
   Stage-2 页表基地址：Y
   翻译粒度、地址宽度、权限配置：……
   ```

   这些信息共同构成后续翻译所需的 translation context，并不会直接给出 `IOVA → PA` 的结果。

4. **它与 Translation Table Walk 是“先找规则，再按规则翻译”。**

   ```text
   DTI Translation Request
            ↓
   Configuration Table Walk
   “应该使用哪套翻译配置？”
            ↓
   Translation Table Walk
   “按照这套配置，输入地址翻译成什么地址？”
            ↓
   GPT Walk（需要时）
   “目标物理 granule 属于哪个 PAS？”
            ↓
   Translation result
   ```

5. **Walk 不代表每次都读取内存。**

   TCU 通常会缓存 configuration 信息。如果所需的 `STE`、`CD` 或相应信息已经命中缓存，就不必重新访问内存执行完整 walk；只有缓存未命中等情况下，才需要读取 configuration tables。

### 一句话总结

> **Configuration Table Walk 根据 StreamID/SubstreamID 找到“用哪套翻译规则”，Translation Table Walk 再按照这套规则计算“地址翻译到哪里”。**

---

## 二、Granule Protection Table（GPT）Walk

### 问题

如何理解下面这句话？

> Granule Protection Table (GPT) walk, that returns a set of GPT tables that are used to check the physical address space with which each granule is associated.

### 核心结论

**TCU 在做地址翻译时，除了查询“地址怎么翻译”，还可能需要查询“这个物理地址属于哪个安全物理地址空间（PAS）”。GPT Walk 就是为了完成这个检查。**

### 解析

1. **GPT 是 Granule Protection Table。**

   它不是普通页表，不负责 `IOVA → PA`；它描述某个 physical granule 属于哪个 Physical Address Space。这里的 granule 可以先理解成物理内存按照固定粒度划分的小块。

2. **GPT Walk 和 Translation Table Walk 解决不同问题。**

   ```text
   Translation Table Walk：
   “这个 IOVA 最终翻译成哪个 PA？”

   GPT Walk：
   “这个 PA 所在的 physical granule 属于哪个 PAS？”
   ```

3. **PAS 可以暂时理解成物理内存的“安全世界归属”。**

   在支持 Arm RME 的系统中，Physical Address Space 不只是 Secure 和 Non-secure，还可能包括 Realm、Root 等 PAS。因此，系统除了确认目标 PA，还必须检查发起访问所使用的安全身份与目标 granule 的 PAS 是否匹配。

   ```text
   请求希望以 Realm 身份访问
                  ↓
   目标 PA 的 granule 是否属于 Realm PAS？
                  ↓
        匹配后才能允许访问
   ```

4. **一次完整 translation 可以粗略理解为：**

   ```text
   DTI Translation Request
            ↓
   Configuration Table Walk
   “这个 StreamID 使用什么 translation context？”
            ↓
   Translation Table Walk
   “IOVA → PA 是什么？”
            ↓
   GPT Walk（需要时）
   “这个 PA granule 属于哪个 PAS？”
            ↓
   检查通过
            ↓
   Translation result 返回 TBU
   ```

   TCU 中存在相应 cache，因此并不是每次 translation 都需要从内存执行完整的 GPT Walk；缓存命中可以减少甚至避免 walk。

### 一句话总结

> **Translation Table 决定“地址在哪里”，GPT 决定“这块物理内存在安全意义上属于谁”；GPT Walk 就是 TCU 查询物理 granule 的 PAS 归属。**

---

## 三类 Walk 的关系

```text
Configuration Table Walk
“使用哪套翻译规则？”
          ↓
Translation Table Walk
“地址最终在哪里？”
          ↓
GPT Walk（需要时）
“目标物理内存在安全意义上属于谁？”
```

> **Configuration Table Walk 找规则，Translation Table Walk 找地址，GPT Walk 查安全归属。**
