## Zookeeper


### 简介

Zookeeper是由Yahoo公司开发的开源的分布式一致性解决方案。它能够提供分布式锁、分布式协调服务、发布/订阅、队列等等服务，Yahoo开发Zookeeper的目的是为集群提供一个统一通用的分布式协调服务，能够保证集群服务的高可用。


### 分布式系统的挑战

 在分布式环境下，我们经常会遇到拜占庭问题、CAP定理等经典。
    
 + 拜占庭问题其实就是信道的可靠性问题，可以归为以下两类问题：
    
	+ 信道可用性
	+ 消息安全性
    
  信道可用性。通常来说，我们集群构建内网环境下，资源和环境基本上是可控的，可以认为内部集群机器之间的通信是没有问题的。
  
  消息安全性。分布式服务之间经常会通过RPC服务，在各个进程之间进行消息通信，防止消息欺骗的方案常用到的为加密和签名。加密是数据内容进行加密，签名可以有效的验证消息的完整性。
  
+ CAP定理。

CAP定理描述的是在分布式环境下，不可能同时保证分区容错性、一致性、服务可用性等三个方面。在分布式环境下，首先要解决的是分区容错性，能够压榨的主要在一致性和服务的可用性方面。因此，出现了Base理论等。Base理论描述的是基本可用和最终一致性，如何理解呢？基本可用的主要思路是需要对服务分级，在服务遇到问题的时候进行降级，保证最基本服务，满足业务的最低要求，保证业务能够进行。最终一致性描述的是，允许数据出现中间状态，但是最终是一致的。


### 数据模型

#### 数据视图树
Zookeeper的视图结构看上去就是Unix文件系统，树形结构，就像文件树一样。树种数据节点成为ZNode，ZNode是Zookeeper的最基本的最小的数据存储单元。父子级的ZNode之间和Linux文件系统一样，采用斜杠（“/”）分割的路径表示，开发人员可以在节点中写入数据，也可以给一个节点创建子节点。

#### ZNode
ZNode基本上可以分为持久性节点、临时性节点、顺序性节点等三大类。但是这三大类并不是互斥的，节点是否持久性和是否有顺序相互组合产生了以下4种类型的节点：

  + 持久节点：该节点创建后就会一直存在于Zookeeper服务器上，直到有删除操作主动删除这个节点。
   
  + 持久顺序节点：该节点在持久性上表现和持久节点一致，差别在于顺序性上。ZNode可以为它的直接子节点维护一份顺序，记录直接子节点创建顺序。Zookeeper在创建子节点时会自动的给子节点名加上一个数字后缀，作为一个新的，完整的节点名（这个节点后缀的上线是整型的最大值）。
  
  + 临时节点：临时节点的生命周期和客户端的Session绑定在一起。如果客户端的Session失效了，那么这个节点就会自动删除掉。Zookeeper规定不能使用临时节点创建子节点，临时节点只能作为叶子节点。
    
  + 临时顺序节点：临时顺序节点就是临时节点上增加了顺序性。

#### ZNode的状态

Zookeeper为了保证一致性，需要维护很多的状态。如果你熟悉MVCC的话，这些状态就很容易理解，Zookeeper为各种类型的操作都打上版本号（事务id）。下面的表格是ZNode的状态信息：


| 属性 | 说明|
|:----:|:-----:|
| czxid | 该节点创建时的事务id，即Created ZXID |
| mzxid | 该节点修改时的事务id，即Modified ZXID |
| ctime | 该节点创建的时间 |
| mtime | 该节点修改的时间 |
| version | 数据节点的版本号 |
| cversion | 子节点的版本号 |
| aversion | 节点的ACL版本号 |
| ephemeralOwner | 如果是临时节点，就是创建该节点的SessionID，如果是持久节点，该值就是为0 |
| dataLength | 数据的长度 |
| pzxid | 节点的子节点列表最后一次修改的事务id，注意：只有更改了子节点列表，才会修改pzxid，修改子节点内容是不会修改pzxid的|

### Zookeeper集群中角色

Zookeeper集群包含了Leader、Follower、Observer等三种角色。Zookeeper通过Leader和Follower完成Leader节点选举和事务决议投票等操作。下面分别对各个角色详细说明。

  + Leader：Leader服务器是整个Zookeeper集群工作机制中的核心，其主要工作有以下两个：
      
      + 事务请求的唯一调度和处理者，保证集群事务处理的顺序性。
      + 集群内部各个服务器的调度者。
      
  + Follower：Follower服务器是Zookeeper集群状态的跟随者，其主要工作为一下三个：
     
      1. 处理客户端非事务请求，转发事务请求给Leader服务器。
      2. 参与事务请求Proposal的投票。
      3. 参与Leader选举投票。
      
  + Observer：Observer角色是3.3开始引入的一个全新的角色，充当观察者，监控Zookeeper集群的最新状态，并且将这些状态同步过来。Observer服务器在工作原理上和Follower非常相似，对于非事务的请求，可以独立处理，将事务请求转发给Leader服务器处理。它和Follower区别主要在于，Observer不参与任何形式的投票，包含事务请求Proposal的投票和Leader选举投票。
   

### 核心原理

Zookeeper的核心是ZAB协议。ZAB协议可以认为是一个简化版本的Paxos算法，虽然ZAB协议的设计者极力否认ZAB是一种Paxos方案。

#### ZAB协议中的角色

  + Leader： 负责所有事物请求的协调处理，为了保证并发时和全局唯一的事物编号，设计为一个。
  + Follower：参与投标和Leader选举，并且处理非事物性请求，路由转发事物请求给Leader
  + Observer：和Follower非常相似，但是不参与Leader选举和投标。

#### ZAB协议

ZAB协议中，所有事物请求必须由一个全局的服务器来协调处理，这样的服务器被称为Leader，而余下的其他服务器则成为Follower服务器。Leader服务器负责将一个客户端事物请求转换成一个事物Proposal（提议），并将该Proposal分发给集群中所有的Fllower服务器。之后Leader服务器需要等待所有Follower服务器的反馈，一旦超过半数的Follower服务器进行了正确的反馈后，那么Leader就会再次向所有的Follower服务器分发Commit消息，要求其将前一个Proposal进行提交。

+ 两模式

  ZAB协议包含崩溃恢复和消息广播两种模式：
  
  + 崩溃恢复模式：
  
	  当整个服务框架在启动过程中，或者当Leader服务器出现网络中断、崩溃退出与重启等异常情况时，ZAB协议就会进入恢复模式，在此模式中选举产生新的Leader。当选举产生了新的Leader服务器，同时集群中已经有过半的机器与Leader服务器完成了状态同步之后，ZAB协议就会退出恢复模式。
	  
   + 消息广播模式：
   
   当集群中已经有过半的Follower服务器完成了和leader服务器状态同步，那么整个服务框架就会进入消息广播模式了。

+ 三阶段

 ZAB协议主要细分为发现、同步、广播等三个阶段。发现阶段主要是Leader选举的过程；同步阶段是在Leader选举成功之后，各个Follower与Leader同步数据的过程；广播阶段是在过半的Follower与Leader同步完状态后，Leader接受客户端事务请求进行处理的过程。
 
 + 发现阶段
   
     1. Follower F将自己的最后接受的事务Proposal的epoch值发给准Leader L。准Leader是 F的最后事务的事务编号的机器的编号。
     2. 准Leader们接受Follower发来的epoch消息，当准Leader L接受到过半的epoch消息后，L会生成newEpoch消息发给这些过半的Follower。newEpouch=max(epouch)+1
     3. 当Follower收到来自准Leader L的newEpouch消息后，进行检验：如果Fllower F的epoch < newEpouch，将epoch设置newEpoch，并且向Leader L发送Ack消息。
     4. 当Leader L收到来自过半的Follower的Ack消息之后，Leader L机会从这些过半的Follower中选出事务编号最大的一个，作为事务的初始化集合I[e]。


  + 同步阶段:
   	
   	   在完成发现阶段之后，就进入同步阶段。
   		
	 1. Leader L会将新的newEpoch值和新的事务集合I[e]以消息的形式发给Quorum中所有的Follower。
	 2. 当Follower F接受到来自Leader L的消息之后，F会校验自己当前最大的事务epoch值是否和L发来的newEpoch值相等，如果相等则利用L发来的初始化事务集合I[e]初始化F的事务集合，初始化成功后向Leader发送ack消息；如果F的epoch值和L发来的newEpoch不相等，说明该Follower尚未处于同步阶段，Follower继续下一轮循环。
	 3. 当Leader接受到过半的Follower针对newEpoch和I[e]的反馈消息后，将会向所有的Follower发送Commit消息。
	 4. 当Follower收到来自Leader的Commit消息后，就会依次处理并提交所有在I[e’]未处理的事务。

 
 + 广播阶段

     1. Leader L接受到客户端新的事务请求后，会生成对应的事务Proposal，并且根据ZXID的顺序向所有的Follower发送提案<e’,<v,z>>，其中epoch(z) = e’
     2. Follower根据消息接受的先后顺序来处理这些来自Leader的事务Proposal，并将它们追加到h[f]中去，之后反馈给Leader。
     3. 当Leader接受到过半的Follower针对事务Proposal<e’,<v,z>>的Ack消息后，机会发送Commit<e’,<v,z>>消息给所有的Follower，要求它们进行事务提交。
     4. 当Follower F接受到来自Leader的Commit<e’,<v,z>>消息后，就会开始提交事务Proposal<e’,<v,z>>。
     
     注意：此时该Follower F必定已经提交了事务<v’,z’> ，其中<v’,z’> 属于h(f)，z’ <= z




### 使用场景

1. 分布式协调器。 对于分布式服务，Master/Slaver模式的设计方案中，经常需要保证Master节点的高可用，热备在节点崩溃时快速切换，这种方案是Zookeeper使用最多的场景。比如Hadoop的nameNode的HA，HBase的HMaster的HA.
2. 发布/订阅。Zookeeper的数据模式是由ZNode组成的树形结构，对于一个ZNode的子节点，当这个ZNode节点状态发生改变的时候，可以向其子节点发送消息。因此可以完成发布订阅的功能。
