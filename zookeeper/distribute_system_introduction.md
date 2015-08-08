# 分布式系统

## 一、分布式的特点

+ 分布性。 分布式系统中多台计算机都会在空间上随意分布，同时，机器的分布情况也会随时变化。
+ 对等性。分布式系统中的计算机没有主／从之分，既没有控制整个系统的主机，也没有被控制的从机，组成不是系统的所有计算机节点都是对等的。
+ 并发行。在一个分布式系统中，多个节点可能会并发的操作一些共享资源，诸如数据库或分布式存储。
+ 缺乏全局时钟。一个典型的分布式系统是由一系列在空间上随意分布的多个进程组成的，具有明显的分布性，这些进程之间通过交换消息进行相互通信，因此在分布式系统中，很难定义两个事件究竟谁先谁后。
+ 故障总是会发生。组成分布式系统的所有计算机，都有可能发生任何形式的故障。一个被大量工程实践所检验过的黄金理论是：任何在设计阶段考虑到的异常情况，一定会在系统实际运行中发生，并且，在系统运行过程中还是会遇到很多在设计时未能考虑到的异常故障。所以，除非需求指标允许，在系统设计时不能放过任何异常情况。
