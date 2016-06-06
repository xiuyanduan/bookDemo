Hadoop被设计用来处理很大量的数据。通常认为这些数据已经存储在HDFS，或者可以大量复制。然而，很多系统不满足这些假设。这些系统产生大量的数据流需要使用Hadoop结构化、存储、分析，Apache Flume就是被设计用来做这些工作的。

Flume被设计用来将大量数据驱动的数据传入Hadoop，典型应用场景是使用Flume收集银行web服务器的日志，然后将这些日志通常新的汇总文件传入到HDFS处理。通常的传输目的地(在Flume中的sink)是HDFS。然而，Flume足够灵活也能够写入到其他系统，例如HBase和Solr。

为了使用Flume，需要运行Flume _agent_端，这是一个Java的常驻进程，运行_sources_和_sinks_，连接_channels_。Flume中的_sources_产生_events_并将它们传送到_channel_，_channel_会在存储这些_events_直到它们被送到sink中。可以认为_source-channel-sink_结合是一个基本的Flume组成部分。

Flume的安装由收集分布式拓扑结构中运行的客户端组成。处于系统边缘的客户端（例如web服务器）收集数据，转发到负责汇总的客户端，最后存储到最终目的地。指定的_sources_和_sinks_客户端被配置用来运行收集工作，实际上使用Flume就是将这个配置放到一起的实践。本文将描述如何搭建Flume拓扑作为Hadoop生态圈的一部分

# 安装Flume

从[官网](http://flume.apache.org/download.html)选择一个稳定版本的可执行压缩包下载Flume，在合适的位置解压tar包：
```
% tar xzf apache-flume-x.y.z-bin.tar.gz
```
配置环境变量：
```
% export FLUME_HOME=~/sw/apache-flume-x.y.z-bin
% export PATH=$PATH:$FLUME_HOME/bin
```
Flume客户端可以使用`flume-ng`命令启动，如下所述。

# 一个例子

为了显示Flume如何工作，让我们从以下设置开始：

1. 追踪本地文件目录的新文本文档
1. 发送文件新增的每一行到数据流

现在手动增加文件，但很容易假设一个进程（例如web服务器）不断产生新文件需要被Flume摄取。在生产环境中，不仅仅是记录文件，还需要通过后来的处理将这些内容写入到HDFS——下文会详述。

在本例中，Flume客户端运行一个单独的_source-channel-sink_，通过一个Java properties文件配置。配置文件决定了使用_sources_的类型、_sinks_和_channels_，它们是互相关联的。如下例所示：
```
agent1.sources = source1
agent1.sinks = sink1
agent1.channels = channel1
agent1.sources.source1.channels = channel1
agent1.sinks.sink1.channel = channel1
agent1.sources.source1.type = spooldir
agent1.sources.source1.spoolDir = /tmp/spooldir
agent1.sinks.sink1.type = logger
agent1.channels.channel1.type = file
```
客户端的层次结构属性名在最顶端。在本例中，只有一个叫做__agent1__的客户端。客户端不同组件的名称在下一层级设置，例如_agent1.sources_描述了在__agent1__上运行的_sources_（本例是一个单独的_sources_，__source1__）。类似地，__agent1__也有_sink_（__sink1__）和_channel_（__channel1__）。

每一个组件的属性在下一层次结构设置，属性的配置根据属性不同而可用。在本例中__agent1.sources.source1.type__被设置为__spooldir__，这是一个spooling directory source监控新文件的spooling目录。spooling directory source定义了__spoolDir__属性，完整的键值是__agent1
.sources.source1.spoolDir__。source的channel由__agent1.sources.source1.channels__设置。

_sink_是一个__logger__，记录事件到输出，它必须和channel（通过__agent1.sinks.sink1.channel property__设置）连接。channel是一个__file__channel，意味着在channel中的事件会永久保存到磁盘中，整个系统的说明如下图所示

在运行例子之前，我们需要在本地文件系统上新建spooling目录：
```
mkdir /tmp/spooldir
```
使用`flume-ng`命令启动Flume客户端：
```
% flume-ng agent \
--conf-file spool-to-logger.properties \
--name agent1 \
--conf $FLUME_HOME/conf \
-Dflume.root.logger=INFO,console
```
如上例中Flume的属性文件需要__-conf-file__指定，客户端的名字必须通过__--name__指定（因Flume可以设置多个客户端，需要指定哪个运行）。__--conf__参数告知Flume寻找它的配置文件，与环境变量类似。

在一个新的终端，在spooling目录内新建一个文件，假设这个文件不可改变。为了阻source读取并改写文件，将内容写入到隐藏文件中。再将文件重命名使source可以读取到：
```
% echo "Hello Flume" > /tmp/spooldir/.file1.txt
% mv /tmp/spooldir/.file1.txt /tmp/spooldir/file1.txt
```
客户端终端的后台，可以看到Flume已经探测到并处理该文件
```
Preparing to move file /tmp/spooldir/file1.txt to
/tmp/spooldir/file1.txt.COMPLETED
Event: { headers:{} body: 48 65 6C 6C 6F 20 46 6C 75 6D 65 Hello Flume }
```
spooling目录将文件按行切割来摄取，每行均产生Flume事件。事件有一个可选的头部和二进制的正文，文档的编写格式为UTF-8。正文部分被sink用十六进制和字符串的形式记录。上文放到spooling目录下的文件只有一行，故只有一个事件在本例中被记录。可以看到文件被soucre重命名为_file1.txt.COMPLETED_，意味着Flume已经处理过该文件，且不会再处理

# 事务和可靠性

Flume将*source*传送到*channel*中，从*channel*传送到*sink*的过程中使用分享的事务。上文所述的例子中，spooling目录的source文件中的每一行产生了一个事件。只有事务成功提交之后，source才会将文件标记为完成

## Batching（定量？）

# The HDFS Sink

## 文件格式
通常来讲，使用二进制格式来存储数据是一个更好的主意，因为它比文本形式占用更少的空间。对HDFS sink来说，文件存储的格式由*hdfs.fileType*和其它的一些参数共同决定

*hdfs.fileType*的默认值为*SequenceFile*，将事件写入到sequence file中,*LongWritable*包括事件的时间(如果*timestamp*头部未设置，则包括当前时间戳），*BytesWritable*值包括事件主体。将*hdfs.writeFormat*设置为*Text*后，可以用Text Writable代替BytesWritable写入到sequence file中

# Fan Out

*Fan out*是将事件从一个source传输到多个channels，使它们能达到多个sinks的术语。例如下述配置，可以将事件传送到HDFS sink（通过channel1a传到sink1a）和一个日志sink（channel1b传到sink1b）
```
agent1.sources = source1
agent1.sinks = sink1a sink1b
agent1.channels = channel1a channel1b
agent1.sources.source1.channels = channel1a channel1b
agent1.sinks.sink1a.channel = channel1a
agent1.sinks.sink1b.channel = channel1b
agent1.sources.source1.type = spooldir
agent1.sources.source1.spoolDir = /tmp/spooldir
agent1.sinks.sink1a.type = hdfs
agent1.sinks.sink1a.hdfs.path = /tmp/flume
agent1.sinks.sink1a.hdfs.filePrefix = events
agent1.sinks.sink1a.hdfs.fileSuffix = .log
agent1.sinks.sink1a.hdfs.fileType = DataStream
agent1.sinks.sink1b.type = logger
agent1.channels.channel1a.type = file
agent1.channels.channel1b.type = memory
```
关键改变在于source被配置成传输到多个channels，通过将**agent1.sources.source1.channels**设置成一个channel names之间用空格分隔的列表 ，本例中为channel1a和channel1b。现在，传到logger sink（channel1b）的channel是一个memory channel，因为我们仅为了做测试传输日志事件，并不关心客户端重启时丢失的事件。同样和前述例子相同，每个channel配置一个sink，如下图所示：

## Delivery Guarantees（传输保证）
Flume从spooling directory source到每一个channel使用分离的事务传送定量的事件。在本例中，通过channel传到HDFS sink使用一个事务，另一个事务传送相同的事件量到logger sink的channel。如果这两个事务有任何一个失败了（例如一个channel已满），则事件将从sources中移出，过段时间再重试。

在本例中，因为我们不在乎是否有事件没有传送到logger sink，所以可以将它的channel设置为一个*optional*的channel，这样如果和它相关的事务失败了，不会导致事件留在source并重试。（注意如果客户端在两个事务均提交完成之前宕机，有关的事件会在客户端重启之后重新传输，即使未提交的事务channel被标记为*optional*）为了达到这个目的，设置source中的*selector.optional*属性，值为用空格分割的channels列表
```
agent1.sources.source1.selector.optional = channel1b
```

> # near-real-time indexing
给事件加索引是实践中使用fan out的一个很好示例。一个单独的事件source被发送到HDFS sink（主要的事件仓库，故使用了一个必需的channel）和一个Solr（或者Elasticesarch）sink，建立一个搜索索引（使用可选的channel）。
MorphlineSolrSink将fileds从Flume事件提取出来将传输到一个Solr文档（使用一个Morphline配置文件），然后载入到一个实时Solr搜索服务中。这个处理过程称作*near real time*，因为只需要几秒就可以将数据处理并展示到搜索结果中。

## 复制和多路选择器
在通常的fan-our流中，事件被复制到所有的channels——但是更多选择是更可取的，以至一些事件被发送到某个channel，其它事件被发送到其它channel。这可以通过设置source的*multiplexing*选择器实现，也能定义路由规则引导指定的事件头部到channels中，参见[官方文档](http://flume.apache.org/FlumeUserGuide.html)

# 分布式：Agent Tiers
如果设置大规模Flume客户端？如果有一个客户端在每一个节点产生新的原始数据，到目前为止的配置，任何时刻每个文件都从一个节点持续性写入到HDFS。如果能够将事件从一组节点聚合到一个文件会更好，这样会产生更少更大的文件（伴随着减少HDFS的压力，并且更有效的处理MapReduce）。同样，如果有必要，文件可以更频繁的回滚因为被更大数量的节点提前数据，导致了从一个事件的建立到可提供分析之间的时间间隔。

将Flume客户端事件聚合是由Flume客户端的*tiers*实现的。第一个*tier*收集原始sources（例如web服务器），将它们发送到第二个*tier*的更小的客户端集合，第二*tier*在写入HDFS之前将第一个*tier*的事件聚合。如果source节点足够多，则需要更多的*tiers*

*Tiers*使用一个特殊的sink将事件通过网络发送，一个对应的*source*接收事件。*Avro sink*通过*Avro RPC*将事件发送到运行在另一个Flume客户端的*Avro source*。也有一个*Thrift sink*通过*Thrift RPC*与一个*Thrift source*协同做同样的事。

> 不要被名字困扰：*Avro sinks*和*source*不能够写入（或读取）*Avro files*。它们只用来在客户端的*tiers*分发事件，并且为了这样做它们使用*Avro RPC*沟通（注意此处用词）。如果需要将事件写入到*Avro files*，使用HDFS sink

下列展示了two-tier Flume配置。该配置文件中有两个客户端，分别叫agent1和agent2。一个类型为agent1的客户端运行在第一个tier，有一个*spooldir*源和一个*Avro sink*通过一个文件channel连接。agent2运行在第二个tier，有一个*Avro source*监听**agent1's**的*Avro sink*发送事件的端口。**agent2**的sink使用相同的HDFS sink配置，如上例（The HDFS Sink章节例子）所示

注意在同一台机器上有两个file channels运行，它们被配置指向不同的数据和检查目录（默认在用户的家目录下）。因此，它们不试图将各自的文件写入到对方中。

```
####A two-tier Flume configuration using a spooling directory source and an
HDFS sink
# First-tier agent
agent1.sources = source1
agent1.sinks = sink1
agent1.channels = channel1
agent1.sources.source1.channels = channel1
agent1.sinks.sink1.channel = channel1
agent1.sources.source1.type = spooldir
agent1.sources.source1.spoolDir = /tmp/spooldir
agent1.sinks.sink1.type = avro
agent1.sinks.sink1.hostname = localhost
agent1.sinks.sink1.port = 10000
agent1.channels.channel1.type = file
agent1.channels.channel1.checkpointDir=/tmp/agent1/file-channel/checkpoint
agent1.channels.channel1.dataDirs=/tmp/agent1/file-channel/data
# Second-tier agent
agent2.sources = source2
agent2.sinks = sink2
agent2.channels = channel2
agent2.sources.source2.channels = channel2
agent2.sinks.sink2.channel = channel2
agent2.sources.source2.type = avro
agent2.sources.source2.bind = localhost
agent2.sources.source2.port = 10000
agent2.sinks.sink2.type = hdfs
agent2.sinks.sink2.hdfs.path = /tmp/flume
agent2.sinks.sink2.hdfs.filePrefix = events
agent2.sinks.sink2.hdfs.fileSuffix = .log
agent2.sinks.sink2.hdfs.fileType = DataStream
agent2.channels.channel2.type = file
agent2.channels.channel2.checkpointDir=/tmp/agent2/file-channel/checkpoint
agent2.channels.channel2.dataDirs=/tmp/agent2/file-channel/data
```
如下图所示：

每一个客户客户端独立运行，使用相同的**--conf-file**配置文件，但是不同的客户端**--name**变量：
```
% flume-ng agent --conf-file spool-to-hdfs-tiered.properties --name agent1 ...
```
和
```
% flume-ng agent --conf-file spool-to-hdfs-tiered.properties --name agent2 ...
```

## 传输保证

# Sink Groups

# Integrating Flume with Applications

# Component Catalog

# Further Reading
