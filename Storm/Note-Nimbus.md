# Nimbus

##Nimbus的作用

Nimbus在Storm集群中主要负责两个功能：

1. 对Topology的任务进行分配调度。
2. 接受用户的命令并且做相应的处理，例如Topology的提交（submit），杀死（kill），激活（active），暂停（deactivate）以及重新调度(rebalance)。

##Nimbus的原理

Nimbus是基于Thrift实现的，它使用了Thrift的THsHaServer服务。THsHaServer即半同步半异步服务模式，它使用一个单独的线程来处理网络I/O，使用一个独立的工作者线程来处理消息，这大大提高了消息的并发处理能力。

出了主服务线程之外，Nimbus中还有一个计时器线程，它的主要作用有3个，具体如下所示：

+ 调用mk-assignment方法启动新一轮的任务分配，调用do-cleanup方法清理Storm元数据。这两项操作每隔NIMBUS-MONITOR-FREQ-SECS（默认值为10s）执行一次。
+ 调用clean-inbox方法清理Nimbus本地目录中Topology的jar包。该操作则每隔NIMBUS-CLEANUP-INBOX-FREQ-SECS（默认值是600s）执行一次。
+ 执行Topology的状态转移事件。这一事件则只有当Nimbus接收到对应的服务请求（如kill、rebalance、activate和deactivate）时才会触发。

###mk-assignments

mk-assignaments方法主要负责对当前及群众所有Topology进行新一轮的任务调度，它一方面会检查已经运行的Topology所占用资源，判断它们是否有问题，是否需要重新分配；另一方面也会根据系统当前的可用资源，为新提交的Topology分配任务。然后，mk-assignments方法会将所有的分配信息保存到Zookeeper中，Supervisor会周期性地检查这些分配信息，并根据这些分配信息做相应的调度处理。

### do-cleanup

该方法用于判断哪些Topology需要清理，并对需要清理的Topology进行相应的操作。do-cleanup会首先删除这些Topology保存在Zookeeper中心跳及错误信息，然后尝试清除Nimbus本地目录中的相关文件，并且从Nimbus心跳缓存中移除对应的信息。