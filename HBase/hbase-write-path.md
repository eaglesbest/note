## HBase的写流程

Apache HBase 是基于HDFS的分布式数据库。HBase使得对HDFS的随机访问和更新更成为可能。但是HDFS中的文件仅仅允许附加操作（apppend），而且文件在创建后是不可变的。你也许会问，HBase是如何提供低延迟的读写功能的？在这篇文章中，我们解析一下HBase的写流程——HBase中的数据是如何更新的。

写流程是指HBase完成的put或delete操作。这个流程从Client开始，转移到region server上（RPC服务），到最后数据完整的写入到HFile中。HBase的写流程包含了在Region Server故障时防止数据丢失的特性。所以，理解写流程可以洞悉HBase的原生数据丢失的防范机制。

每个HBase表由三类服务依托与管理：

 + 一个活跃的master server
 + 一个或多个备份的master server
 + 多个region server

 Region server 控制HBase的表。因为HBase的表可能很大，它们会被分成多个region分区。每个region server处理一个或者多个region。需要注意的region server是服务于HBase的表数据的唯一服务，一台主服务器崩溃不会导致数据丢失。

 HBase的数据组织和sorted map相似。也是将key排序后，切分到不同的分片(region server)或分区（region）中。HBase client 通过调用put或delete命令来更新表。
