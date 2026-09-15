P2：

​	1.SMMU是什么？

  为系统中除CPU外的其他设备提供地址翻译服务的组件，并在地址翻译的过程中对地址空间的访问权限以及memory attribute进行检查和管理。

​	2.stage1 only translation是怎么样的，stage2 only translation是怎么样的？

​     stage1 only tanslation是VA到PA的直接转换，对应是当一个device被分配给hypervisor时，hypervisor访问该设备时使用的是stage1 only tanslation

​     stage2 only translation是在hypervisor管理下的IPA到PA的转换，当一个device被分配给一个Guest OS时，Guest OS用物理地址直接访问这个device，它会发送IPA，然后IPA在hypervisor的管控下被转换为PA。



P6：

   1.context的概念：

​     不同设备/进程所使用的地址翻译环境，包含地址转换的页表以及一些地址翻译过程中用到系统设置。

   2.SMMUv3的架构演化特点：

​      相比于前代架构，能支持大量的不同进程/设备的地址转译环境，而非固定数量的地址转译环境；

   地址转译所用到的配置信息存放在memory中，而非存放在寄存器中

   3.TODO：RME的引入对SMMU的转译行为会有什么影响

P14：

​    1.stream与stream ID的理解：

stream是来自不同设备的地址转译请求，stream ID是对各个转译请求的设备来源标识

P15：

​     1.substream的理解：

​         一个设备的地址空间下存在多个进程地址空间，substream是这些子进程发出的地址转译请求

​     2.subtream的地址转译特点：

​        一个设备下的所有substream共享一个stage 2  tranlation

​         这些substream有都拥有独立的stage 1 translation

P16~P17：

​       如果SMMU有设置TrustZone的支持，SMMU在接收stream时首先会对stream的安全状态进行检查，secure与non-secure stream所使用的地址转译环境下的配置table是不同的，secure与non-secure stream通过SEC_SID去识别。

P19

​     1.Fault指的是地址转译过程中遇到的异常返回结果，可以时遇到了无效的tale walk结果（translation fault）、访问权限不允许（access permission fault）、输入地址不在系统配置的范围中（Address Size fault）

​     2.SMMU处理fault的两种方式
​	 一种是直接宣告地址访问失败，向上游的发起方返回终止信息，这个过程不会引入软件的修复

​         另一种是不立马宣告访问终止，而是先将transaction pending起来，然后产生中断给CPU，让软件介入修复，如遇到页表缺失的场景，中断会通知软件重建页表，修复完成之后，软件通过COMMAND Queue通知SMMU修复完成，SMMU就可以完成后续的transaction访问了

P20

​      1.区分三种bypass：

​      stream的STE被配置bypass，这种情况下，SMMU仍要lookup STE，根据STE上config字段才判断的transaction bypass，这种情况下，STE中的MTCFG、SHCFG、ALLOCFG字段还会决定是否override transaction的memory attributes

​    global bypass：SMMU被disable，根据SMMU\_(S)\_GBPA 去override memory attribute

   physical bypass: 地址不会被转译，memory attribute不会被override，通过设置axMMUVALID =0实现的。Physical bypass的原因通常是由于发出地址请求已经是PA，不需要进行地址翻译了，但穿过SMMU的过程中，如果SMMU支持RME，SMMU仍然会对该物理地址做GPC的检查操作。

​     2.memory attribute属于软件的强制行为，与系统的一致性拓扑结构有关，与地址转译行为无关

P22

​    1.TODO（描述方式）SMMU的编程接口有哪些，分别是什么用途

​	寄存器 --- 提供STE、CD等配置表的起始位置以及大小，以及SMMU系统功能的支持

| 寄存器组          | 典型寄存器                                 | 核心作用                                                     |
| ----------------- | ------------------------------------------ | ------------------------------------------------------------ |
| 能力发现          | `SMMU_IDR0~IDR5`、`SMMU_AIDR`、`SMMU_IIDR` | “这颗 SMMU 支持什么？”是否支持 PRI？ 是否支持 ATS？ 是否支持 Stall model？ 支持多大的 StreamID？ 支持多大的 SubstreamID？ 支持 4KB / 16KB / 64KB granule？ 最大 PA/OAS 多大？ 是否支持 16-bit VMID？ 是否支持 RME？ |
| 全局控制          | `SMMU_CR0/CR0ACK`、`CR1`、`CR2`            | “SMMU 开不开、Queue 开不开、内部访问属性怎么配？”            |
| Global Bypass     | `SMMU_GBPA`                                | SMMU disabled 时 bypass transaction 的属性                   |
| Stream Table      | `SMMU_STRTAB_BASE`、`SMMU_STRTAB_BASE_CFG` | “Stream Table 在内存哪里、有多大、什么格式？”                |
| Command Queue     | `SMMU_CMDQ_BASE/PROD/CONS`                 | 软件向 SMMU 下维护命令                                       |
| Event Queue       | `SMMU_EVENTQ_BASE/PROD/CONS`               | SMMU 向软件报告 fault/event                                  |
| PRI Queue         | `SMMU_PRIQ_BASE/PROD/CONS`                 | PCIe PRI Page Request                                        |
| 中断              | `SMMU_IRQ_CTRL/ACK`、各种 `*_IRQ_CFG*`     | Event/PRI/GERROR 怎么通知 CPU                                |
| Global Error      | `SMMU_GERROR/GERRORN`                      | SMMU 自己的管理基础设施错误                                  |
| Secure/Realm/Root | `SMMU_S_*`、`SMMU_R_*`、`SMMU_ROOT_*`      | Secure、Realm、GPT/GPC 等独立资源                            |

配置的表 ----地址转译的信息以数据结构的形式存放在memory

translation table --- 存放地址转译的结果

消息传递的通道 --- CMDQ、EVTQ、PRIQ等提供软件与SMMU的通信渠道，用于传递维护命令、反馈事件以及发送页表请求