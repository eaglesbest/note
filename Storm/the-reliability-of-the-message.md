## 消息的可靠性保证

`Storm`提供了几种不同的消息可靠处理的级别。包含`best effort`、`at least once`和`exactly once`（`exactly once`是通过Trident来实现的）。

### 消息的"完整处理"是什么意思

 + tuple树（或者 tuple DAG）

  一个从spout中发送出去的tuple会产生许多基于它创建的tuple。从而构成了以spout tuple为根(root)的一颗tuple树（早期Storm只能只能tuple tree,现如今可以支持到DAG，这里说tuple DAG更合适）。如下图：

  ![storm tuple tree](../charts/storm_tuple_tree.png)


 + 完全处理

  在storm的，tuple完全处理是指，spout发送出去的tuple，经过bolt衍生出来的tuple树，如果tuple树中的每个tuple消息都得到正确处理(成功ack了)，这样就说明了spout发送出来的tuple得到完全处理了。

  对应的，如果在指定的时间内tuple树中的消息没有完成处理或者失败，这就意味这个tuple处理失败。

  关于tuple消息处理超时时间，可以通过`Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS`参数在构造topology进行配置，默认时间为30s。

### 消息完整处理或处理失败后会发送什么？

  为了更好的理解消息完整处理的意义，我们先了解一下来自spout的tuple的生命周期。

  + spout tuple 生命周期

   下面是spout接口的定义：

   ```
   public interface ISpout extends Serializable {
       void open(Map conf, TopologyContext context, SpoutOutputCollector collector);
       void close();
       void nextTuple();
       void ack(Object msgId);
       void fail(Object msgId);
   }
   ```

   首先，通过调用Spout的nextTuple方法。Storm向Spout请求一个tuple。Spout会使用open方法中提供的SpoutOutputCollector向它的一个输出数据流中发送一个tuple。在发送tuple的时候，Spout会提供一个消息id，这个id会在后续的过程中用语识别tuple。

   随后，tuple会被发送到对应的bolt中去，在这个过程中，Storm会很小心地跟踪创建的消息树。如果Sorm检测到某个tuple被完整的处理，Storm会根据Spout提供消息id调用最初发送tuple的Spout任务的ack方法。对应的，Storm在监测到tuple超时之后就会掉用fail方法。

   注意，对一个特定的tuple，ack和fail都只会由最初创建这个tuple的任务执行。也就是说，即使Spout在集群中有很多个任务，某个特定的tuple也只会由创建它的任务（而不是其他的任务）来处理成功或失败的结果。

+ 完整处理和失败会发生什么？

  在此，我们以官网的KestrlSpout为例来看看消息的可靠处理中Spout做了什么。在KestrlSpout从Kestrel队列中取出一条消息时，可以看作它“打开”了这条消息。也就是说，这条消息实际上并没有从队列中真正地取出来，而是保持着一个“挂起”状态，等待消息处理完成的信号。在挂起状态的消息不会被发送到其他的消费者中。另外，如果消费者（客户端）端开了连接，所有处于挂起状态的消息都会重新放回队列中。在消息“打开”的时候Kestrel会给客户端同时提供消息体数据和一个唯一的id。KestrelSpout在使用SpoutOutputCollector发送tuple的时候就会把这个卫衣的id当作消息id，一段时间后，在KestrelSpout的ack或者fail方法调用的时候，KestrolSpout就会通过这个消息id向Kestrel请求消息从队列中移除（对应ack的情况）或者将消息重新放回到队列(对应的fail的情况).

### Storm的可靠性API

 使用Storm的可靠性机制的时候你需要注意两件事：

  1. 在tuple树中创建新节点连接时必须通知Storm；
  2. 在每个tuple处理结束的时候也必须向Storm发出通知。

通过这两个操作，Storm就能够监测到tuple树会在何时完成处理，并适时地调用ack或者fail方法。Strom的API提供了一种非常精确的方式来实现这两个操作。

Storm中指定tuple树中的一个连接成为“锚定”(anchoring)。锚定是在新发送的tuple的同时发生的。让我们以下面的Bolt为例说明这一点，这个Bolt将一次包含橘子的tuple分割成若干的单词tuple。

```
   public class SplitSentence extends BaseRichBolt {

          OutputCollector _collector;

          public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
              _collector = collector;
          }

          public void execute(Tuple tuple) {
              String sentence = tuple.getString(0);
              for(String word: sentence.split(" ")) {
                  _collector.emit(tuple, new Values(word));
              }
              _collector.ack(tuple);
          }

          public void declareOutputFields(OutputFieldsDeclarer declarer) {
              declarer.declare(new Fields("word"));
          }
  }
```
通过将输入 tuple 指定为 emit 方法的第一个参数，每个单词 tuple 都被“锚定”了。这样，如果单词 tuple 在后续处理过程中失败了，作为这棵 tuple 树的根节点的原始 Spout tuple 就会被重新处理。相对应的，如果这样发送 tuple：

```
_collector.emit(new Values(word));
```

就称为“非锚定”。在这种情况下，下游的 tuple 处理失败不会触发原始 tuple 的任何处理操作。有时候发送这种“非锚定” tuple 也是必要的，这取决于你的拓扑的容错性要求。

一个输出 tuple 可以被锚定到多个输入 tuple 上，这在流式连接或者聚合操作时很有用。显然，一个多锚定的 tuple 失败会导致 Spout 中多个 tuple 的重新处理。多锚定操作是通过指定一个 tuple 列表而不是单一的 tuple 来实现的，如下面的例子所示：

```
List<Tuple> anchors = new ArrayList<Tuple>();
anchors.add(tuple1);
anchors.add(tuple2);
_collector.emit(anchors, new Values(1, 2, 3));

```

多锚定操作会把输出 tuple 添加到多个 tuple 树中。注意，多锚定也可能会打破树的结构从而创建一个 tuple 的有向无环图（DAG），如下图所示：

![storm tuple DAG](../charts/storm_tuple_dag.png)

Storm的程序实现既支持对树的处理，同样也支持对DAG的处理（由于早期的 Storm 版本仅仅对树有效，所以“tuple树”的这个糟糕的概念就一直沿袭下来了）。

锚定其实可以看作是将tuple树具象化的过程——在结束对一棵tuple树中一个单独tuple的处理的时候，后续以及最终的tuple都会在Storm可靠性API的作用下得到标定。这是通过OutputCollector的ack和fail方法实现的。如果你再回过头看一下SplitSentence的例子，你就会发现输入tuple是在所有的单词tuple发送出去之后被ack的。

你可以使用OutputCollector的fail方法来使得位于tuple树根节点的Spout tuple立即失败。例如，你的应用可以在建立数据库连接的时候抓取异常，并且在异常出现的时候立即让输入tuple失败。通过这种立即失败的方式，原始Spout tuple就会比等待tuple超时的方式响应更快。

每个待处理的tuple都必须显式地应答（ack）或者失效（fail）。因为 Storm是使用内存来跟踪每个tuple的，所以，如果你不对每个tuple进行应答或者失效，那么负责跟踪的任务很快就会发生内存溢出。

Bolt处理tuple的一种通用模式是在execute方法中读取输入tuple、发送出基于输入tuple的新tuple，然后在方法末尾对tuple进行应答。大部分Bolt都会使用这样的过程。这些Bolt大多属于过滤器或者简单的处理函数一类。Storm 有一个可以简化这种操作的简便接口，称为BasicBolt。例如，如果使用BasicBolt，SplitSentence的例子可以这样写：

```
public class SplitSentence extends BaseBasicBolt {
        public void execute(Tuple tuple, BasicOutputCollector collector) {
            String sentence = tuple.getString(0);
            for(String word: sentence.split(" ")) {
                collector.emit(new Values(word));
            }
        }

        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("word"));
        }
}
```

这个实现方式比之前的方式要简单许多，而且在语义上有着完全一致的效果。发送到BasicOutputCollector的tuple会被自动锚定到输入tuple上，而且输入tuple会在execute方法结束的时候自动应答。

相对应的，执行聚合或者联结操作的Bolt可能需要延迟应答tuple，因为它需要等待一批tuple来完成某种结果计算。聚合和联结操作一般也会需要对他们的输出tuple进行多锚定。这个过程已经超出了IBasicBolt的应用范围。
