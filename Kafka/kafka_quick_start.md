# 快速入门（二）

----

### 一、下载

[下载](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.2.0/kafka_2.10-0.8.2.0.tgz) 0.8.2.0发布版本，下载后解压

	tar -xvf kafka_2.10-0.8.2.0.tgz
	cd kafka_2.10-0.8.2.0
	


### 二、启动服务

***启动Zookeeper***

Kafka使用了Zookeeper作为分布式协调，因此首先应该启动Zookeeper服务。你也可以使用下面这个方便的脚本，启动Kafka包中单节点的Zookeeper实例。

	bin/zookeeper-server-start.sh config/zookeeper.properties
	
***现在来启动Kafka服务***

启动Kafka服务需要指定服务的配置文件。前面是启动的shell文件，后面跟上服务配置文件

	bin/kafka-server-start.sh config/server.properties
	
还有一种启动方式，就是作为后台进程启动，在上面的命令后面加上一个&符号，如下:

	bin/kafka-server-start.sh config/server.properties &
	
	
	
### 三、创建主题
	
创建主题时，需要指定复制因子（replication-factory）、分区数量、主题名称、zookeeper服务。

举个栗子。现在我们创建一个只有一个分区，并且复制因子为1，名称为“test”的主题，使用本地的zookeeper服务协调broker

	bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 partitions 1 --topic test

现在，我们可以执行如下的命令，查看主题列表：

	bin/kafka-topics.sh --list --zookeeper localhost:2181
	
此外，除了按照以上的方式创建主题以外，如果不存在发布的主题时，你也可以配置brokers自动创建主题。

### 四、发送消息

Kafka可以通过一个命令行客户端从文件或者标准输入终端中发送内容作为消息到Kafka集群中，默认情况下，每一行将会被分割为一条消息。

栗子：运行producer，通过控制台发送一些消息到Kafka服务器中

	bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test 
	This is a message
	This is another message
	
### 五、启动消费者

Kafka也允许通过命令行把消息导入到标准输出中终端上。

	bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
	
如果你用不同的终端运行上面的命令（发送消息和启动消费者），那么你可以在生产者终端上输入消息，在消费者终端上看到消息被消费。

所有命令行工具有更多的选项，不带参数运行该命令会显示使用信息，记录更多的细节。

### 六、安装多broker的集群

之前运行的Kafka只有一个broker，但是这个没有意思，现在将我们的集群扩展到3个节点，安装多个broker实例。还是在我们的本地机器上来做这件事。

1.	我们需要为每个broker配置好一个配置文件。

		cp config/server.properties config/server-1.properties
		cp config/server.properties config/server-2.properties
	
2.	接下来，编辑这些新文件，并且设置下列各项属性：

		config/server-1.properties:
	    	broker.id=1
	    	port=9093
	    	log.dir=/tmp/kafka-logs-1
 
		config/server-2.properties:
	    	broker.id=2
	    	port=9094
	    	log.dir=/tmp/kafka-logs-2
    	
	集群中每个节点的broker.id属性是唯一的。我们可以替换端口和日志目录。
	
3. 我们已经启动了一个单节点的Zookeeper实例，所以我们现在要再启动两个新的Zookeeper节点。

	撸代码

		bin/kafka-server-start.sh config/server-1.properties &
		bin/kafka-server-start.sh config/server-2.properties &
	
4.	现在新建一个主题（topic），设置改主题的复制因子为3。
		
		bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factory 3 --partitions 1 --topic my-replicated-topic

	现在我们有一个集群了，那么broker是如何工作的呢？去看看运行“describe topics”命令：
		
		bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
	
	> Topic:my-replicated-topic	PartitionCount:1	ReplicationFactor:3	Configs:
	
	>Topic: my-replicated-topic	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
	
	这里是输出的解释。第一行给出的所有分区的摘要，每增加一个线给出关于一个分区的信息。因为我们只有一个分区这个话题，只有一条线。
	
	+ leader负责该节点的所有读取和给定的分区写入。每个节点将成为领导者对分区中的随机选择的部分。
	+ replicas是复制日志此分区节点列表，无论他们是否是领导或者即使他们目前还活着。
	+ isr是一组“in-sync”副本。这是副本列表是目前活着，抓行动领导的子集。

	***注意：在我们的例子中节点1是主题唯一分区的leader***

	我们能够对已经创建的主题运行下面类似的命令：
		
		bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
		
	>Topic:test	PartitionCount:1	ReplicationFactor:1	Configs:
	
	>Topic: test	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
	
5. 发布消息

	让我们向新主题中发布一些消息：
	
		bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
	
	 
		
	
	




	
	
