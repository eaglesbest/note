#YARN的初识

## Yarn出现背景

在Hadoop1.X（MR1）中，JobTracker肩负了资源管理和作业控制等两部分工作，这样造成了JobTracker负载过重。从设计的角度来说，未能将资源管理相关功能和应用程序相关功能分开，造成了扩展性、资源利用率和多框架支持存在不足，难以支持多种计算框架。因此，出现了MR2。

MR2（第二代MapReduce框架）的基本设计思想是将JobTracker的资源管理和作业控制（作业监控、监控）分拆成两个独立的进程。让资源管理和具体的应用程序无关，它只负责整个集群的资源（内存、CPU、磁盘）管理，而作业控制进程则是与应用程序相关的模块，且每个作业控制进程只负责管理一个作业。

+ MR2的功能模块:
	+ YARN专管理资源和调度
	+ ApplicationMaster负责与具体的应用程序相关的任务切分、任务调度和容错。

## YARN基本架构

+ ResourceManager：全局资源管理器，负责整个系统的资源管理和分配。
+ ApplicationMaster：应用程序控制进程，负责单个应用程序的管理。

YARN 总体上仍然是 Master/Slave 结构,在整个资源管理框架中,ResourceManager 为 Master,NodeManager 为 Slave,ResourceManager 负责对各个 NodeManager 上的资源进行 统一管理和调度。当用户提交一个应用程序时,需要提供一个用以跟踪和管理这个程序的 ApplicationMaster,它负责向 ResourceManager 申请资源,并要求 NodeManger 启动可以占 用一定资源的任务。由于不同的 ApplicationMaster 被分布到不同的节点上,因此它们之间 不会相互影响。

### YARN中的基本组件

1. ####ResourceManager(RM)
   RM是一个全局的资源管理器，负责整个系统的资源管理和分配。它的主要构成：调度起（Scheduler）和应用程序管理器（Applications Manager, ASM）。
	+ 调度器：

		调度器根据容量、队列等限制条件(如每个队列分配一定的资源,最多执行一定数量的作业等),将系统中的资源分配给各个正在运行的应用程序。需要注意的是,该调度器是 一个“纯调度器”,它不再从事任何与具体应用程序相关的工作,比如不负责监控或者跟踪 应用的执行状态等,也不负责重新启动因应用执行失败或者硬件故障而产生的失败任务, 这些均交由应用程序相关的 ApplicationMaster 完成。调度器仅根据各个应用程序的资源需 求进行资源分配,而资源分配单位用一个抽象概念“资源容器”(Resource Container,简 称 Container)表示,Container 是一个动态资源分配单位,它将内存、CPU、磁盘、网络等 资源封装在一起,从而限定每个任务使用的资源量。此外,该调度器是一个可插拔的组件, 用户可根据自己的需要设计新的调度器,YARN 提供了多种直接可用的调度器,比如 Fair Scheduler 和 Capacity Scheduler 等。
	+ 应用程序管理器
	
		应用程序管理器负责管理整个系统中所有应用程序,包括应用程序提交、与调度器协商资源以启动 ApplicationMaster、监控 ApplicationMaster 运行状态并在失败时重新启动它等。

2. ####ApplicationMaster(AM)

	用户提交的每个应用程序均包含一个 AM,主要功能包括: ❑ 与 RM 调度器协商以获取资源(用 Container 表示);
	
	+ 将得到的任务进一步分配给内部的任务;	+ 与 NM 通信以启动 / 停止任务;	+ 监控所有任务运行状态,并在任务运行失败时重新为任务申请资源以重启任务。
		当前YARN自带了两个AM实现,一个是用于演示AM编写方法的实例程序distributedshell,它可以申请一定数目的 Container 以并行运行一个 Shell 命令或者 Shell 脚本 ; 另一个是运行 MapReduce 应用程序的 AM— MRAppMaster,我们将在第 8 章对其进行介 绍。此外,一些其他的计算框架对应的 AM 正在开发中,比如 Open MPI、Spark 等。

3. ####NodeManager(NM)

	NM是每个节点上的资源和任务管理器,一方面,它会定时地向 RM 汇报本节点上的 资源使用情况和各个 Container 的运行状态;另一方面,它接收并处理来自 AM 的 Container 启动 / 停止等各种请求。

4. ####Container

	Container 是 YARN 中的资源抽象,它封装了某个节点上的多维度资源,如内存、 CPU、磁盘、网络等,当 AM 向 RM 申请资源时,RM 为 AM 返回的资源便是用 Container 表示的。YARN 会为每个任务分配一个 Container,且该任务只能使用该 Container 中描述的 资源。需要注意的是,Container 不同于 MRv1 中的 slot,它是一个动态资源划分单位,是 根据应用程序的需求动态生成的，使用了轻量级资源隔离机制 Cgroups 进行资源隔离。



