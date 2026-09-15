# Split-stage ATS 的功能限制：Fully Coherent Cache 与 PCIe P2P

记录日期：2026-09-03  
记录端：本地  
来源：ChatGPT 对话 `SMMU`，对培训 slide 中 Split-stage ATS limitations 的追问（截图时间点未知）  
状态：Inbox；Fully Coherent 的协议条件、PCIe ACS 路由规则及平台 P2P 拓扑待结合 SMMUv3、AMBA Coherency 与 PCIe 规范校验

## 问题

培训 slide 所说的 Split-stage ATS 限制——`Devices cannot be fully coherent` 和 `peer-to-peer PCIe ACS traffic still requires stage 2 translation`——分别是什么意思？为什么这两个限制都与 Device 拿不到最终 PA 有关？

## 回答

### 核心结论

> **Split-stage ATS 的安全性来自“最终 PA 留在 SMMU Stage 2 手里”，而它的功能限制也来自同一件事：Device ATC 只有 IPA。Device Cache 无法直接用 PA 响应系统 snoop，PCIe Fabric 也不能直接用 IPA 完成基于物理地址的 P2P 路由。**

## 1. `Device cannot be fully coherent` 的含义

这里的 Fully Coherent 是指 Device 内部存在 cache，并能像 CPU coherent cache 一样参与系统硬件一致性协议，正确响应针对 physical cache line 的 snoop。

系统一致性通常围绕最终 Physical Address 建立：

```text
CPU Cache：   PA 0x8000_1000
                 ↕ snoop
Device Cache：PA 0x8000_1000
```

Device Cache 必须知道自己是否保存了被 snoop 的同一条 physical cache line。

这与泛化的“DMA coherent”或软件无需手工 flush 的平台属性不能简单等同；这里关注的是 Device 自身缓存直接参与 PA-based hardware snoop coherency。

## 2. Split-stage ATS 为什么限制 Fully Coherent Device Cache

Split-stage ATS 只把 Stage 1 结果交给 Device：

```text
Device ATC：IOVA 0x1000 → IPA 0x4000

SMMU Stage 2：IPA 0x4000 → PA 0x8000_1000
```

Device Cache 若只知道 IPA：

```text
Device Cache tag：IPA 0x4000
System snoop：    PA  0x8000_1000
                         ↓
                    无法直接匹配
```

因此在该模型下，Device 无法只凭 Split-stage ATS 返回的地址建立与系统 PA snoop 空间完全一致的 physically addressed coherent cache。

Full ATS 则把最终 PA 返回 Device：

```text
ATC：IOVA → PA
Device Cache tag：PA
System snoop：PA
              ↓
            可匹配
```

所以 Full ATS 更适合需要直接掌握 Physical Address 的 Fully Coherent Device Cache。

## 3. PCIe P2P 与 ACS 是什么

PCIe Peer-to-Peer（P2P）允许一个 Endpoint 直接访问另一个 Endpoint，例如：

```text
          PCIe Switch
          /         \
       GPU A       GPU B

理想数据路径：GPU A → PCIe Switch → GPU B BAR
```

它的价值是避免 transaction 绕行 Host Root Port/SMMU。

ACS（Access Control Services）是 PCIe Switch/Port 的访问控制与路由能力，可根据系统策略决定 P2P traffic：

- 允许直接送往 Peer。
- 或强制 Redirect 到 Root Port。

因此 slide 中的 `peer-to-peer PCIe ACS traffic` 可理解为受 ACS 策略控制的 Endpoint-to-Endpoint P2P traffic。

## 4. Split-stage ATS 为什么让 P2P 仍需要 Stage 2

假设 GPU B BAR 的系统物理地址为：

```text
PA = 0xA000_0000
```

而 GPU A 在 Split-stage ATS 下只获得：

```text
IPA = 0x5000_0000
```

二者关系仍需 SMMU Stage 2：

```text
IPA 0x5000_0000 → PA 0xA000_0000
```

如果 GPU A 直接把 IPA 发往 PCIe Switch：

```text
GPU A
  │ Address = IPA
  ▼
PCIe Switch
  │ 基于 transaction address 匹配目标 BAR
  ▼
无法直接得到真正的 PA/BAR 路由结果
```

因为 direct P2P 没有经过 SMMU，IPA→PA 无人完成。因此 Split-stage 场景必须在某个合适位置执行 Stage 2；典型概念路径是把 transaction Redirect 回 Root Port/SMMU：

```text
GPU A → PCIe Switch → Root Port → SMMU Stage 2 → 目标路径
```

这不表示 Split-stage ATS 绝对不能使用 P2P，而是：

> **仅凭 Device ATC 中的 IPA，无法直接完成不经过 Host Root Port 的 physical-address-based P2P routing。**

实际路由还受 PCIe 拓扑、BAR 编址、ACS policy、Root Complex 和平台安全策略约束。

## 5. Full ATS 对 P2P 的优势

Full ATS 让发起 Device 直接获得目标 PA：

```text
GPU A ATC：IOVA → GPU B BAR PA
        ↓
GPU A 发送 PA
        ↓
PCIe Switch 根据物理地址路由
        ↓
GPU B BAR
```

这使 direct P2P 更自然，但并不代表 ACS 一定允许该 transaction 直达 Peer，也不代表安全检查可以省略；平台仍需根据信任和访问控制策略决定路由。

## 6. 两个限制的共同根因

```text
                 Split-stage ATS
                        │
               Device ATC 只有 IPA
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
Device 不知道最终 PA          PCIe Fabric 没有最终 PA
          │                           │
无法直接构建 PA-tagged         无法直接进行 PA-based
Fully Coherent Cache           Peer-to-Peer routing
          │                           │
限制 Fully Coherent            P2P 仍需某处执行 Stage 2
```

Split-stage ATS 的设计取舍是：

- **安全收益**：最终 PA 与 Stage 2 permission 仍由 SMMU/Hypervisor 掌握。
- **功能代价**：Device 无法直接使用 PA 完成某些 coherency 和 P2P 能力。

## 7. 为什么 DPT 被描述为补偿方案

DPT 的目标是让 Device 继续使用 Full ATS 的最终 PA，从而保留：

- PA-based Fully Coherent Device Cache。
- 更灵活的 direct PCIe P2P routing。

同时通过：

```text
SID → STE → VMID
PA → DPT → permitted VMID
             ↓
           权限比较
```

补充 Device/VM 级访问控制。这就是培训材料所说的“结合 Split-stage ATS 的安全性与 Full ATS 的灵活性”的含义。

## 面试版总结

> **Split-stage ATS 只把 Stage 1 的 IPA 交给 Device，最终 PA 留在 SMMU Stage 2。由于系统 snoop 和 PCIe BAR 路由最终面向 physical address，Device 只有 IPA 时，无法直接建立 PA-tagged、可响应系统 snoop 的 Fully Coherent Cache，也无法仅凭该地址完成绕过 Root Port 的 direct PCIe P2P routing；P2P traffic 仍需某处执行 Stage 2。它的安全优势和功能限制都来自同一个设计选择：Device 拿不到最终 PA。**

## 一句话总结

> **Split-stage ATS 把 PA 留给 SMMU，因此更安全；但 Device 没有 PA，也就难以直接参与 PA-based full coherency 和 direct PCIe P2P routing。**

## 关联记录

- [Full ATS 与 Split-stage ATS](40_Full_ATS与Split-stage_ATS.md)
- [DPT 为 Full ATS 补充设备权限检查](41_DPT为Full_ATS补充设备权限检查.md)
- [Full ATS 的安全边界](36_Full_ATS的安全边界.md)
- [SMMU 与 I/O Device 的三种集成方式](39_SMMU三种设备集成方式.md)

