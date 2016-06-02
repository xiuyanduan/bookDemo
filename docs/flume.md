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
客户端的层次结构属性名在最顶端。在本例中，只有一个叫做**agent1**的客户端。客户端不同组件的名称在下一层级设置，例如_agent1.sources_描述了在**agent1**上运行的_sources_（本例是一个单独的_sources_，**source1**）。类似地，**agent1**也有_sink_（**sink1**）和_channel_（**channel1**）。

每一个组件的属性在下一层次结构设置，属性的配置根据属性不同而可用。在本例中**agent1.sources.source1.type**被设置为**spooldir**，这是一个spooling directory source监控新文件的spooling目录。spooling directory source定义了**spoolDir**属性，完整的键值是**agent1
.sources.source1.spoolDir**。source的channel由**agent1.sources.source1.channels**设置。

_sink_是一个**logger**，记录事件到输出，它必须和channel（通过**agent1.sinks.sink1.channel property**设置）连接。channel是一个**file**channel，意味着在channel中的事件会永久保存到磁盘中，整个系统的说明如下图所示

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
如上例中Flume的属性文件需要**-conf-file**指定，客户端的名字必须通过**--name**指定（因Flume可以设置多个客户端，需要指定哪个运行）。**--conf**参数告知Flume寻找它的配置文件，与环境变量类似。

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
