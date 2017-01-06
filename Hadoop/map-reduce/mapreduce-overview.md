## mapreduce概览

### Mapper

Mapper将输入键/值对映射到一组中间键/值对。

Maps是将输入记录转换为中间记录的各个任务。变换的中间记录不需要与输入记录具有相同的类型。给定的输入对可以映射到零个或许多输出对。

Hadoop MapReduce框架为由作业的InputFormat生成的每个InputSplit生成一个map任务。

总的来说，Mapper实现通过Job.setMapperClass（Class）方法传递Job作业。然后框架为该任务的InputSplit中的每个键/值对调用map（WritableComparable，Writable，Context）。然后，应用程序可以覆盖清除（Context）方法以执行任何所需的清除。

输出对不需要与输入对具有相同的类型。给定的输入对可以映射到零个或许多输出对。通过对context.write（WritableComparable，Writable）的调用收集输出对。

应用程序可以使用计数器报告其统计信息。

与给定输出键相关联的所有中间值随后由框架分组，并传递到缩减器以确定最终输出。用户可以通过Job.setGroupingComparatorClass（Class）指定Comparator来控制分组。

Mapper输出被排序，然后按照Reducer进行分区。分区的总数与作业的reduce任务的数量相同。用户可以通过实现自定义分区程序来控制哪些键（从而记录）转到哪个Reducer。

用户可以选择通过Job.setCombinerClass（Class）指定一个组合器，以执行中间输出的本地聚合，这有助于减少从Mapper传输到Reducer的数据量。

中间排序的输出始终以简单（key-len，key，value-len，value）格式存储。应用程序可以控制是否以及如何压缩中间输出以及如何通过配置使用CompressionCodec。


#### Map任务数量

map任务的数量通常由输入的总大小，即输入文件的块的总数来驱动。

对于map任务的并行性的正确水平似乎是每节点大约10-100个map任务，但是对于非常cpu-light映射任务已经被设置为300个map任务。 任务设置需要一段时间，因此最好是map任务需要至少一分钟才能执行。

因此，如果你期望10TB的输入数据和128MB的块大小，你将最终有82,000map，除非Configuration.set（MRJobConfig.NUM_MAPS，int）（它只提供框架的提示）用于设置 它甚至更高。

### Reducer

Reducer减少了一些中间值，它们共享一个键值给一组较小的值。

作业的缩减数由用户通过Job.setNumReduceTasks（int）设置。

总的来说，Reducer实现是通过Job.setReducerClass（Class）方法为作业传递Job，并且可以覆盖它来初始化自己。 然后框架为分组输入中的每个<key，(list of values)>的pair调用reduce（WritableComparable，Iterable <Writable>，Context）方法。 然后，应用程序可以覆盖清除（Context）方法以执行任何所需的清除。

Reducer有3个主要阶段：shuffle，sort和reduce。

#### Shuffle

Reducer的输入是mapper任务的排序输出。 在这个阶段，框架通过HTTP提取所有mapper任务的输出的相关分区。

#### Sort

在这个阶段，框架使用键来分组Reducer输入（因为不同的mapper可能输出相同的键）。shuffle和sort阶段同时发生; 当map输出正在被获取时，它们被合并。

#### Secondary Sort

如果对分组的中间key等价规则都要求是从那些用于减少前分组key不同，则一个可以指定通过Job.setSortComparatorClass（类）一个比较器。因为Job.setGroupingComparatorClass（类）可以被用来控制key如何中间被分组，这些可以结合使用，以模拟第二次排序上的值。

#### Reduce

在此阶段，为分组输入中的每个<key，(list of values)>pair调用reduce（WritableComparable，Iterable <Writable>，Context）方法。reduce任务的输出通常通过Context.write（WritableComparable，Writable）写入文件系统。

应用程序可以使用计数器报告其统计信息。

Reducer的输出没有排序。

#### Reduce的任务数量

正确的reduce任务数数似乎是0.95或1.75乘以（<节点数> * <每个节点最大容器数<）。使用0.95，所有reduce可以立即启动，并在map完成时开始传输map输出。 使用1.75，更快的节点将完成他们的第一轮reduce，并发起第二波reduce做更好的负载平衡工作。

增加reduce的数量增加了框架开销，但增加了负载平衡并降低了故障成本。

上面的缩放因子略小于整数，以在框架中为推测任务和失败的任务预留几个reduce时隙。

#### Reducer NONE

如果不需要reduce，则将reduce-task的数量设置为零是合法的。

在这种情况下，map-tasks的输出直接到FileSystem，进入由FileOutputFormat.setOutputPath（Job，Path）设置的输出路径。 框架不会在将map输出写入文件系统之前对其进行排序。


#### Partitioner

Partitioner分隔key空间。

Partitioner控制中间map输出的键的分区。 key（或key的子集）通常通过散列函数用于导出分区。 分区的总数与作业的reduce任务的数量相同。 因此，这控制m个reduce任务中的哪一个将中间key（并且因此记录）发送到reduce。

HashPartitioner是默认的Partitioner。

#### Counter

计数器是MapReduce应用程序报告其统计信息的工具。

Mapper和Reducer实现可以使用计数器报告统计信息。

Hadoop MapReduce捆绑了一个通用的mappers，reducers和partitioners库。
