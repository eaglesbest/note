#Storm对接Kafka时网络IO飙高的问题

storm在线上已经运行了几个月了，上线之后一直都是很稳定健康的运行着。突然，有一天，一直业务的Topology没有消费消息。经过日志定位，发现时KafkaSpout不再emit消息了。

对于这样的问题，我们首先把所有的可能性找出来。

1. 代码上有Bug，没有发现，导致对于Kafka的中消息解析出现问题。
2. Kafka出问题了。
3. Kafka上这个Topic出现问题了
4. Zookeeper中的KafkaSpout的元数据出现问题了
5. 消息发生变化了（比如格式或者大小）。
6. 架构上的调整导致的。
7. 其他尚未发现的问题导致的。

把所有可能的问题列出来之后，就是逐一去验证。

  代码Bug。这只业务是上线最早的业务，此前几个月都一直很健康的跑着，如果是代码的问题，必然是最近发布导致的。先查询发布日志，通过发布日志看，最近没有发布。如果是代码对于内容的解析的问题，那么这个问题早就暴露了。

为了进一步确定问题的发生的位置，我们直接从Kafka中取出消息不做任何处理，直接打印日志，看看情况。结果发现日志没有打印出来。这说明并不是Topology中的业务代码导致的。

   Kafka出问题。首先查看Kafka集群的状态，cpu、内存都正常，之后网络的IO很高，3个broker都达到150M/s。我们查看Zoookeeper中各个Broker和partition的信息，这些信息在，没有丢失，而且3个broker没有节点宕机的情况。但是现在Kafka的网络IO很高，我们要确定Kafka的网络IO高是不是这一支引起的，对于其他的业务是不是正常的。我们停掉出现问题的Topology，网络IO迅速下降，下降到12M/s，这是我们的正常情况。如果是Kafka的锅，那么肯定对于其他的Topic也会有影响的，如此看来Kafka暂时不用背锅。

   Kafka上这个Topic出现问题了。对于这个问题，我们先想到的使用Kafka提供的控制台消费者试一下，看看能不能消费数据。我们试了发现是可以正常消费数据的。但是现在情况是这个Topic的业务上线之后网络IO就升高，和这个Topic或者这种类型的消息有关。于是，我们决定创建一个新的topic，将这种类型的消息放入到新的Topic中，通过测试程序发现过了一段时间后，测试Topology也不消费新的topic消息了，问题复现了。由此可以排出topic的问题。问题和消息类型非常有关系了。

  我们通过ZK的客户端查看了和这种类型的消息的Spout的元数据，元数据一切正常。而且我们架构没有变动。
  
  通过上面的几个检查，我们阅读了KafkaSpout的源码，KafkaSpout中使用的SimpleConsumer作为Kafka的消费者。我们跟踪KafkaUtils发现，这个实现存在topic没有新消息，consumer端没有谁的轮询没有设置等待参数，也没有在client线程里进行短暂的sleep，这几乎是死循环似的不断和broker通信。
  
  这总算是定位到问题了。Kafka的maxWait默认是0，当Kafka的服务端没有消息市判断是否阻塞到maxWait，不过我们很快发现与需要给minBytes设置一个大于0的值才可以。参考KafkaApis的handlerFetchRequest方法
  
  ```
  val dataRead = readMessageSets(fetchRequest)
val bytesReadable = dataRead.values.map(_.messages.sizeInBytes).sum

if(fetchRequest.maxWait <= 0 ||
   bytesReadable >= fetchRequest.minBytes ||
   fetchRequest.numPartitions <= 0) {
  ...
  requestChannel.sendResponse(new RequestChannel.Response(request, new FetchResponseSend(response)))
} else {
  ...
  fetchRequestPurgatory.watch(delayedFetch)
}
  ```
  
 于是决定修改KafkaSpout的配置，但是我们很快发现，SpoutConfig并没有提供设置minBytes的接口。阿西吧！！！
 没有办法，我们决定基于官方的KafkaSpout的进行修改。将minBytes设置为默认值为1。
 
 在修改源码的时候，我们也在分析这个问题的发生原因。突然记得前两天客户端采集sdk，单个报文数据太大，导致flume往Kakfa写不进去，为了兼容客户端的bug，我们将Nginx的单个日志最大调整2M，Kafka的单条消息大小调整为5M。然而，我们并没有修改Kafka的Consumer的Fetch操作的Buffer的大小，这样Consumer没有fetch下来那些很大的消息，导致重复fetch。再增加fetchSizeBytes、BufferSizeBytes、SocketTimeOut等配置。具体如下
 
 ```
spoutConfig.bufferSizeBytes = 10 * 1024 * 1024;
spoutConfig.fetchSizeBytes = 5 * 1024 * 1024;
spoutConfig.fetchMaxWait = 5000;
spoutConfig.socketTimeoutMs =30000;
spoutConfig.useStartOffsetTimeIfOffsetOutOfRange = true;
 ```
 
 修改后将Topology提交上去，问题的没有出现了。
 
 本质原因，是Kafka的消息大小变大了，然而我们没有修改Kafka的Fetch的fetchSizeBytes和Buffer的大小。另外，我们在测试的发现，当topic中没有消息的时候，也存在网络IO异常的情况。KafkaSpout并没有考虑这种情况。
 
 