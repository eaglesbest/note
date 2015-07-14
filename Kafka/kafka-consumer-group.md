# Kafka Consumer Group例子

 [参考文献wiki](https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example)

## 使用高级的Consumer的

## 一、为什么使用高级的Cosumer

有时候，从Kafka中读取数据逻辑，并不需要关系如果处理消息的offsets，只是想要数据。因此，高级的Consumer提供抽象许多Kafka的消费事件细节。

首先，高级的Consumer将从特定的分区（partition）上一次读取的offset存储在Zookeeper中。这些offset基于Kafka的Consumer提供的Group存储的。

Consumer的Group名称是贯穿Kafka集群全局的。所以，对于任何老的消费者代码逻辑被新的代码替换的地方都要特别小心。当新的进程使用相同的Consumer Group名称启动时，Kafka将会添加进程的线程设置可用的线程，去消费Topic中的消息，并且触发消费者线程负载均衡。在负载均衡期间，Kafka分配可用的Partition给可用的线程。如果混合一些新老的逻辑，可能需要讲一个分区移动到另外一个进程中。这样可能导致一个消息被老的逻辑使用。

## 二、如何设计一个高级的Consumer

首先要了解的是，高级的Consumer,它可以是（而且应该是）一个多线程的应用程序。对于一个主题的线程数和分区（partition）数量也有一些规则：

* 当提供的线程数量多于partition的数量，则部分线程将不会接收到消息；
* 当提供的线程数量少于partition的数量，则部分线程将从多个partition接收消息；
* 当某个线程从多个partition接收消息时，不保证接收消息的顺序；可能出现从partition3接收5条消息，从partition4接收6条消息，接着又从partition3接收10条消息；
* 当添加更多线程时，会引起kafka做re-balance, 可能改变partition和线程的对应关系。
因为突然停止Consumer以及Broker会导致消息重复读的情况，为了避免这种情况在shutdown之前通过Thread.sleep(10000)让Consumer有时间将offset同步到zookeeper

-----

## 三、应用例子

### Maven依赖

	 <!--Kafka 消息依赖-->
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.10</artifactId>
            <version>0.8.2.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>0.8.2.0</version>
        </dependency>
        
### Consumer线程

	import kafka.consumer.ConsumerIterator;
	import kafka.consumer.KafkaStream;
	import kafka.message.MessageAndMetadata;

	public class ConsumerThread implements Runnable {
		private KafkaStream kafkaStream;
		//线程编号
		private int threadNumber;
		public ConsumerThread(KafkaStream kafkaStream, int threadNumber) {
		  this.threadNumber = threadNumber;
		  this.kafkaStream = kafkaStream;
		}
		public void run() {
		  ConsumerIterator<byte[], byte[]> it = kafkaStream.iterator();
		  StringBuffer sb = new StringBuffer();
		//该循环会持续从Kafka读取数据，直到手工的将进程进行中断
		  while (it.hasNext()) {
		   MessageAndMetadata metaData = it.next();
		   sb.append("Thread: " + threadNumber + " ");
		   sb.append("Part: " + metaData.partition() + " ");
		   sb.append("Key: " + metaData.key() + " ");
		   sb.append("Message: " + metaData.message() + " ");
		   sb.append("\n");
		   System.out.println(sb.toString());
		  }
		  System.out.println("Shutting down Thread: " + threadNumber);
		}
	}
	

### 其余程序

	import kafka.consumer.ConsumerConfig;
	import kafka.consumer.KafkaStream;
	import kafka.javaapi.consumer.ConsumerConnector;
	 
	import java.util.HashMap;
	import java.util.List;
	import java.util.Map;
	import java.util.Properties;
	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	 
	public class ConsumerGroupExample {
	   	private final ConsumerConnector consumer;
	   	private final String topic;
	   	private  ExecutorService executor;
	 
	    public ConsumerGroupExample(String a_zookeeper, String a_groupId, String a_topic) {
	        consumer = kafka.consumer.Consumer.createJavaConsumerConnector(
	                createConsumerConfig(a_zookeeper, a_groupId));
	        this.topic = a_topic;
	    }
	 
	    public void shutdown() {
	        if (consumer != null) consumer.shutdown();
	        if (executor != null) executor.shutdown();
	    }
	 
	    public void run(int a_numThreads) {
	        Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
	        topicCountMap.put(topic, new Integer(a_numThreads));
	        //返回的Map包含所有的Topic以及对应的KafkaStream
	        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
	        List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic);
	 
	        //创建Java线程池
	        executor = Executors.newFixedThreadPool(a_numThreads);
	 
	        // 创建 consume 线程消费messages
	        int threadNumber = 0;
	        for (final KafkaStream stream : streams) {
	            executor.submit(new ConsumerTest(stream, threadNumber));
	            threadNumber++;
	        }
	    }
	 
	    private static ConsumerConfig createConsumerConfig(String a_zookeeper, String a_groupId) {
	        Properties props = new Properties();
	        //指定连接的Zookeeper集群，通过该集群来存储连接到某个Partition的Consumer的Offerset
	        props.put("zookeeper.connect", a_zookeeper);
	       //consumer group 的ID
	        props.put("group.id", a_groupId);
	        //Kafka等待Zookeeper的响应时间（毫秒）
	        props.put("zookeeper.session.timeout.ms", "400");
	       //ZooKeeper 的‘follower’可以落后Master多少毫秒
	        props.put("zookeeper.sync.time.ms", "200");
	      //consumer更新offerset到Zookeeper的时间
	        props.put("auto.commit.interval.ms", "1000");
	 
	        return new ConsumerConfig(props);
	    }
	 
	    public static void main(String[] args) {
	        String zooKeeper = args[0];
	        String groupId = args[1];
	        String topic = args[2];
	        int threads = Integer.parseInt(args[3]);
	 
	        ConsumerGroupExample example = new ConsumerGroupExample(zooKeeper, groupId, topic);
	        example.run(threads);
	         //因为consumer的offerset并不是实时的传送到zookeeper（通过配置来制定更新周期），所以shutdown Consumer的线程，有可能会读取重复的信息
	        //增加sleep时间，让consumer把offset同步到zookeeper
	        try {
	            Thread.sleep(10000);
	        } catch (InterruptedException ie) {
	 
	        }
	        example.shutdown();
	    }
	}

-----	
	
## 四、offset管理

kafka会记录offset到zk中。但是，zk client api对zk的频繁写入是一个低效的操作。0.8.2 kafka引入了native offset storage，将offset管理从zk移出，并且可以做到水平扩展。其原理就是利用了kafka的compacted topic，offset以consumer group,topic与partion的组合作为key直接提交到compacted topic中。同时Kafka又在内存中维护了的三元组来维护最新的offset信息，consumer来取最新offset信息的时候直接内存里拿即可。当然，kafka允许你快速的checkpoint最新的offset信息到磁盘上

## 五、High Level Consumer的管理工具

* 查看指定Group的consumer消费状况使用如下命令：

		bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper zk1.dmp.com:2181,zk2.dmp.com:2181,zk3.dmp.com:2181 --group pv

* 重置offset可以使用如下命令：

		bin/kafka-run-class.sh kafka.tools.UpdateOffsetsInZK earliest config/consumer.properties page_visits

	3个参数， 
	[earliest | latest]，表示将offset置到哪里 
	consumer.properties ，这里是配置文件的路径 
	topic，topic名，这里是page_visits


