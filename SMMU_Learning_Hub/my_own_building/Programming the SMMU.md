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

P26
   1.STE和CD的概况
   STE和CD都是地址翻译所用的配置表项。
   STE根据SID查找出来，STE中记录了主要控制信息包括：
         config  --- 地址的翻译阶段是怎么样的，stage1和stage2的使能情况   
         s1contextptr --- 在stage1使能的情况下，STE还会给出CD的查找位置
         S2TTB --- stage2 页表查找的起始地址
         VMID --- 不同虚拟机的地址空间标识符
         S2T0SZ --- stage2 转译地址的范围
         S2TG  --- 页表的granule
         table walk的属性 --- S2IR0/OR0 规定页表的cacheability S2SH0规定了页表访问的sharedomain
    SSID根据SSID查找出来，记录了stage1 transaltion的配置信息
        TTB0/TTB1：stage1 翻译页表的起始地址
        ASID：进程地址空间的标识符
        T0SZ/T1SZ、TG0/TG1：地址范围和翻译粒度；
        IR/OR/SH：SMMU 访问页表时所用的memory attribute；
        MAIR：记录页表所映射的地址空间的memory attribute。
        
P32
   <重点核心>描述一下SMMU地址翻译的过程 (过程有些复杂，通过图示表达出来) 
  SMMU地址翻译的过程主要包括两个查表的过程，分别配置表的查找和页表的查找
  首先是configuration lookup，配置表包含STE和CD，分别记录了stage2和stage1的地址转译配置，在lookup开始时，SMMU首先通过寄存器获知Stream table的起始地址，然后使用SID去index
出具体的STE表项。STE中记录了stage2 页表查找的起始地址以及CD查找的位置，如果需要进行stage1的翻译，那么就需要通过CD查找的基地址结合SSID找到最终的表项，CD中记录stage1页表查找的起始地址和ASID等地址转译信息
   然后page table 的lookup，SMMU首先会去lookup TLB，如果TLB miss的话就开启table walk，这一阶段和MMU的table walk过程是类似的，结合前面提供的translation context，获得必要的页表基地址，然后结合VA的字段去做index，完成逐级页表的遍历。在多级页表遍历的过程中，我们遇到这几种情况：首先如果是lookup到一个page类型的entry，它会提供下级页表的基地址让我们继续去查找，然后也可能会遇到page或是block类型，说明table walk已经完成，它们提供的是最终的地址转译结果，最后也可能会遇到invalid的表项，这说明发生了translation fault，需要上报给CPU进行处理。

P33、P34、P36
   软件和SMMU通过数据队列进行通信
   CMDQ是软件向SMMU发送维护命令的通道，主要的维护命令有：TLBI、CFGI、CMD_SYNC
   EVENTQ是SMMU向软件反馈特殊事件的通道，反馈的时间包括：
       CERROR_ILL:CMDQ中出现的非法命令
       C_BAD_STREAMID:使用的SID超出了配置的范围
       C_BAD_CD:CD表格无效 valid == 0
P37
  Global Error：影响 SMMU 全局运行的错误：
       CMDQ 错误，且可能使后续命令处理暂停，直到软件清除错误；
       访问 EVTQ 时异常 abort；
       写 MSI 时产生的异常Abort。
  软件通过 SMMU_(S_)GERROR 观察错误状态，通过 SMMU_(S_)GERRORN 的规定方式确认/清除，并可配置 Global Error 中断。
P38、P39
    ATS是什么：PCIe的设备会提前将地址转译的结果放到本地cache中
    PRIQ：设备的 ATS Translation Request 可能因为页面尚未准备好而失败，随后设备发送 PRI Request。

P42 
  SMMU的启动配置是怎么样的：
  1.创建并初始化配置表项stream table，在系统寄存器中配置查找的起始地址
  2.创建并初始化CMDQ、EVENTQ，并在寄存器中设置队列存放的位置、producer指针、consumer指针。PRIQ只有在PRI功能启用时才会创建
  3.通过SMMU_CR寄存器配置stream table和CMDQ的内存访问属性，如cacheability、shareability
  4.配置中断路径，为EVENTQ、PRIQ、global error配置IRQ
  5.设置GBPA以及全局bypass时，memory attribute的输出类型
  6.通过寄存器使能SMMU
