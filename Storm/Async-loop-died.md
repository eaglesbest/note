#Async loop died

今天查看Storm的UI，发现其中又一个Topology的Spout有一些failed。于是找到对应的worker日志。找到了一些异常信息如下：

```
2016-04-01 13:32:46 b.s.util [ERROR] Async loop died!
java.lang.RuntimeException: java.lang.RuntimeException: Remote address is not reachable. We will close this client Netty-Client-storm4.dmp.com/10.10.169.229:6701
        at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:128) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:99) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:80) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.disruptor$consume_loop_STAR_$fn__758.invoke(disruptor.clj:94) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.util$async_loop$fn__457.invoke(util.clj:431) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at clojure.lang.AFn.run(AFn.java:24) [clojure-1.5.1.jar:na]
        at java.lang.Thread.run(Thread.java:745) [na:1.7.0_79]
Caused by: java.lang.RuntimeException: Remote address is not reachable. We will close this client Netty-Client-storm4.dmp.com/10.10.169.229:6701
        at backtype.storm.messaging.netty.Client.connect(Client.java:166) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.messaging.netty.Client.send(Client.java:203) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.utils.TransferDrainer.send(TransferDrainer.java:54) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.daemon.worker$mk_transfer_tuples_handler$fn__5927$fn__5928.invoke(worker.clj:322) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.daemon.worker$mk_transfer_tuples_handler$fn__5927.invoke(worker.clj:320) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.disruptor$clojure_handler$reify__745.onEvent(disruptor.clj:58) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:125) ~[storm-core-0.9.2-incubating.jar:0.9.2-incubating]
        ... 6 common frames omitted
2016-04-01 13:32:46 b.s.util [INFO] Halting process: ("Async loop died!")
```

在网上找到一篇相关内容的[Async loop died文章](http://support.huawei.com/ecommunity/bbs/10242847.html)。

发生这个问题的原因是各个worker之间通过Netty通信，当netty通信连接在设置时间内无法建立，就会导致worker1的netty客户端被关闭，进而导致worker1进程kill掉，并被supervisor重起。尤其是当某拓扑中包含的worker较多，且worker中线程较多，worker进行启动时，需要耗费较长时间时，往往会由于netty连接超时而导致已启动的worker又被重启，导致整个拓扑无法正常部署。

解决方案：加大worker间Nettey建立连接的超时总时间，保证在设定时间内，topo能够稳定部署成功。

1．增大storm.messaging.netty.max_retries设置，默认为300，可设置如600

2．增加storm.messaging.netty.max_wait_ms设置，默认为1000，可设置如2000

哟西！先从github找到[storm.yaml](https://github.com/apache/storm/blob/master/conf/defaults.yaml)的配置文件

在各个节点的conf/storm.yaml增加以下内容：

```
#storm worker netty config
 storm.messaging.netty.max_retries: 300 
 storm.messaging.netty.max_wait_ms: 3000
 storm.messaging.netty.min_wait_ms: 100
```
重启集群，重新提交topology，经过几个小时的监控，再也没有发现类似的问题了。