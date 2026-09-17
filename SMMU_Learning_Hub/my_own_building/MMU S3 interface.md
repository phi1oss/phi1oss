P6
   1.<TODO>ACE-lite TBU interface相关的问题
       1.BIU的作用介绍？
       2.TLBU与BIU的交互使用协议是LTI，转译与数据转发操作在图中的拓扑机构里是怎么进行的
P7~P9
  1.<TODO> 这些属性的具体应用场景是怎么样的
  2.<TODO>AxMMUFLOW、rresp[2]/bresp[2] translation faults指的是什么样的场景，如何管理的
  3.AxMMUVALID --- physical bypass所用的信号，对于一个已经完成转译的transaction，使用该信号去指示SMMU不对其进行地址转译
  4.cache stash ---device在写入数据时携带stash信息，通知系统在进行一致性维护操作时，将cache line的数据allocate到目标组件附近
  5.deallocation --- 读完之后释放cache line
  6.atomic transaction --- 用于将架构中定义的一些原子操作交给下游组件去完成，这些原子操作一般包括读-修改-写三部分，为了保证原子操作的不可分割性，atomic transaction会将操作的过程放到目标数据的附近组件去完成，也就是在读完数据之后，在数据附近的ALU完成修改过程，然后更新到数据的存放位置。
  7.loopback --- 用于标识当前transaction ID对应transaction并不唯一，通过Axloop标识进一步区分这些相同ID的transaction
  8.unique ID --- 传递信息：用于标识当前transaction ID对应transaction是唯一的
  9.poison --- 指示传递的数据中已经发现了错误
  10.read data chunking ---在一笔transaction中，将需要读取的数据分成多个chunk，这些chunk会配置顺序的标识，这些chunk在返回时可以不按照存放的顺序返回，在接收端可以将原本的数据顺序还原出来。该特性所优化的场景是靠前的chunk返回较慢可以先接收靠后的chunk。
  11.MTE --- 用于解决内存访问安全问题，程序使用的地址指针可能被前面数组的溢出操作所改写，导致后面跳到恶意程序进行执行，MTE的做法是给地址指针加上一个tag，并给地址所在的内存位置也加上一个tag，这两个tag的关系是一一对应的，如果指针的值被修改，后续程序运行到指针所在的位置，拿修改后的指针去访问内存数据，tag值因为不匹配，就可以检查出访问的错误。
  
  
