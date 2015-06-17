# 快速入门（二）

----

### 第一步：下载

[下载](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.2.0/kafka_2.10-0.8.2.0.tgz) 0.8.2.0发布版本，下载后解压

	tar -xvf kafka_2.10-0.8.2.0.tgz
	cd kafka_2.10-0.8.2.0
	
----

### 第二步：启动服务

***启动Zookeeper***

Kafka使用了Zookeeper作为分布式协调，因此首先应该启动Zookeeper服务。你也可以使用下面这个方便的脚本，启动Kafka包中单节点的Zookeeper实例。

	bin/zookeeper-server-start.sh config/zookeeper.properties
	
***现在来启动Kafka服务***

启动Kafka服务需要指定服务的配置文件。前面是启动的shell文件，后面跟上服务配置文件

	bin/kafka-server-start.sh config/server.properties
	
还有一种启动方式，就是作为后台进程启动，在上面的命令后面加上一个&符号，如下:

	bin/kafka-server-start.sh config/server.properties &
	
----
	
### 创建主题
	
创建主题时，需要指定复制因子（replication-factory）、分区数量、主题名称、zookeeper服务。

举个栗子。现在我们创建一个只有一个分区，并且复制因子为1，名称为“test”的主题，使用本地的zookeeper服务协调broker

	bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 partitions 1 --topic test

现在，我们可以执行如下的命令，查看主题列表：

	bin/kafka-topics.sh --list --zookeeper localhost:2181
	
