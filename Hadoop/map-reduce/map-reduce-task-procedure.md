#map-reduce任务过程详解

在编写一个简单的`MapReduce`作业时，只需要实现`map()`和`reduce()`两个函数即可。一旦将作业提交到集群上后，Hadoop内部会将这两个函数封装导Map Task和Reduce Task中，同时将它们调度到多个节点上并行执行，而任务执行过程中可能涉及到数据跨节点传输，记录按key分组等操作均由Task内部实现好了，用户无须关心。

## Map Task

 + Read
 + Map
 + Collect
 + Spill
 + Combine

  对于Map Task而言，它的执行过程可以概述为：首先，通过用户提供的InputFormat酱对应的InputSplit解析成一系列的key/value，并依次交给用户编写的map()函数处理；接着按照指定的Partitioner对数据分片，以确定每个key/value将交给哪个Reduce Task处理；之后将数据交给用户定义的Combiner进行一次本地规约（用户没有定义则直接跳过）；最后将处理结果保存到本地磁盘上。
## Reduce Task

 + Shuffle
 + Merge
 + Sort
 + Reduce
 + Write

  对于Reduce Task而言，由于它的输入数据来自各个Map Task，因此首先需通过HTTP请求从各个已经运行完成的Map Task上拷贝对应的数据分片，待所有数据拷贝完成后，再以key关键字你对所有的数据进行排序，通过排序，key相同的记录聚集到一起形成若干分组，
