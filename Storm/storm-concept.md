#Storm中的概念

## topology

Storm分布式计算结构称为topology（拓扑），由stream（数据流）、spout（数据流的生产者），bolt（运算节点）组成。

## stream

Storm的核心数据结构是tuple。tuple是包含了一个或多个键值对的列表，Strean是由无限制的tuple组成的序列。

## spout

spout是一个Storm topology的主要数据入口，充当数据采集器的角色，是topology的数据源，通常spout连接到数据源，比如消息队列、数据库、文件，将数据转化为tuple，并将tuple作为数据流发射到topology的后续节点中。

## bolt

bolt为一个topology的节点，完成具体的功能计算，比如过滤、连接（join）、聚合、结果入库等基本运算功能。bolt可以将一个或者多个数据流作为输入，对数据完成计算后，选择性的输出为一个或者多个数据流。bolt可以订阅多个由spout或者bolt发射的数据流，这样就可以建立复杂的数据流转换网络。
