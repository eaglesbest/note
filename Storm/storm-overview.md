# Storm概览

##Storm集群组成

一个```Storm```集群表面上和```Hadoop```集群非常相似。在```Hadoop```上你可以运行```MapReduce```任务，在```Storm```上你可以运行```topologies```。```MapReduce```任务和```Topologies```有很大的差别，最大的不同在于```MapReuce```任务最终会结束，然而```topology```进程永远不会停止（除非你kill它）。

```Strom```集群有两类节点：```master```节点和```worker```节点。```master```节点运行的守护进程称为```Nimbus```，它和```Hadoop```的```JobTracker```相似。```Nimbus```的职责是给集群分发```topology```代码，给```topology```分配机器，监控失败，对于失败的任务重新分配。

每一个worker节点运行的守护进程称为```Supervisor```。```supervisor```监听为工作分配的机器，启动和关闭```Nimbus```分配给它的工作进程。每一个```worker```进程执行都是一个```topology```的子集；一个运行的```topology```由分布在多台机器的的多个worker进程组成。

```Nimbus```和```Supervisor```之间的所有协调工作是通过```Zookeeper```集群完成的。另外，```Nimbus```守护进程和```Supervisor```守护进程是快速失败并且无状态的；所有的状态是保存在```Zookeeper```或者本地磁盘中，这意味着你可能```kill -9 Nimbus```或者```Supervisors```进程，它们重新启动后就行什么都没有发生一样，这种设计是```storm```非常稳定的原因。

## Topologies

在Storm上做实时计算，你需要创建一个topologies。topology是一个计算图，topology的每一个节点包含处理逻辑，节点之间通过数据链接起来。

运行一个topology是非常简单的，首先，你将所有的代码打包到一个jar包中，然后运行下面的命令

```
storm jar all-my-code.jar backtype.storm.MyToplogy arg1 arg2
```

运行的类为backtype.storm.MyTopology，运行参数为arg1和arg2。在这个类的主函数中定义topology，并且提交给Nimbus。```storm jar ```部分是连接Nimbus并且上传jar文件。

## Streams

```stream```是```Storm```中核心抽象。```stream```是指无限的```tuple```序列。```Storm```提供了将流转化为分布式地、可靠地新流的原语。

```Storm```提供的基本的流转化的原语称为```spouts```和```bolts```。```Spouts```与```bolts```提供了实现和运行你特定业务逻辑的接口。

```spout```是流的源头。```spout```可以从消息队列（数据库...等等）读取```tuple```，并且发射它们作为一个流（stream）。

```bolt```消息任意数量的输入流，完成一些处理逻辑，可能会发射新流。复杂的流转化，像```tweets```的主题趋势计算，需要多个```bolt```多个步骤。```Bolt```能够完成任何功能，如过滤tuples、流数据聚合、流连接（`join`）、数据库交互以及更多。

Spout和Bolt的网络被包装成`topology`，这是您提交`Storm`执行最高级的抽象。`topology`是每个节点由`spout`或`bolt`组成的流转化的图。图的边表示该bolt订阅了哪些流。当`spout`或`bolt`发射`tuple`到流中，这些`tuple`会被发送到每一个订阅了这条流的`bolt`中。

## 数据模型

`Storm`使用tuple作为其数据模型。一个`tuple`是名值列表，`tuple`的域可以是任何类型的对象，`Strom`支持原始类型如：`string`、`byte数组`等作为域的值，如果是其它类型的对象，你需要实现这个类型的序列化。

`topology`的每一个节点必须定义其发射tuple的输出域。比如，下面这个`bolt`中定义了发射的输出域为“double”与“triple”

```
public class DoubleAndTripleBolt extends BaseRichBolt {
    private OutputCollectorBase _collector;

    @Override
    public void prepare(Map conf, TopologyContext context, OutputCollectorBase collector) {
        _collector = collector;
    }

    @Override
    public void execute(Tuple input) {
        int val = input.getInteger(0);        
        _collector.emit(input, new Values(val*2, val*3));
        _collector.ack(input);
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("double", "triple"));
    }    
}
```

declareOutputFields方法定义了当前组建输出域。