# Hadoop集群设置

## 目的

本文描述了如何在几个节点到成千上万节点的环境中安装和配置Hadoop集群。为了更好的使用Hadoop，首先应该在单节点服务器上安装（参考[单节点安装Hadoop](http://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-common/SingleCluster.html)）。

本文未包括进阶主题，例如[Security](http://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-common/SecureMode.html)和高可用。

## 先决条件

  * 安装Java，参考[Hadoop Wiki](http://wiki.apache.org/hadoop/HadoopJavaVersions)
  * 从Apache官方镜像下载一个稳定版本的Hadoop

## 安装

安装一个Hadoop集群包括在所有集群的服务器上解压软件，或者通过适配操作系统的包管理器安装软件。将硬件按照功能来划分很重要。

一般来说，在集群中一台服务器被设计成NameNode，另外一台服务器被专门用作ResourceManager。这些服务器是masters。其他服务（例如Web App Proxy Server和MapReduce Job History server）通常在专用硬件或者根据负载共享运行。

集群中剩下的服务器均是DataNode和NodeManager，它们是slaves。

## Non-Secure模式下配置Hadoop

Hadoop的Java配置包括两类重要的配置文件

  * 默认只读配置 - `core-default.xml`, `hdfs-default.xml`, `yarn-default.xml` and `mapred-default.xml`.

  * Site-specific配置 - `etc/hadoop/core-site.xml`, `etc/hadoop/hdfs-site.xml`, `etc/hadoop/yarn-site.xml` and `etc/hadoop/mapred-site.xml`.

另外，可以通过设置`etc/hadoop/hadoop-env.sh` and `etc/hadoop/yarn-env.sh`中的site-specific值来控制分布式集群bin/目录下的Hadoop脚本。

为了配置Hadoop集群，需要设置Hadoop daemons运行时需要的环境变量

HDFS daemons由NameNode, SecondaryNameNode,和DataNode组成. YARN damones由ResourceManager, NodeManager,和WebAppProxy组成. 如果使用MapReduce，那么MapReduce Job History Server也需要运行。大规模集群下，通常都运行在专门的服务器上。

### 配置Hadoop Daemons运行环境

管理员应用使用 `etc/hadoop/hadoop-env.sh` 并选择性使用`etc/hadoop/mapred-env.sh`和`etc/hadoop/yarn-env.sh` 脚本以设置Hadoop daemon进程的 site-specific环境

至少需要指定`JAVA_HOME`，以保证在每一个远程节点上正常运行。

管理员可以使用下表配置各个daemons


Daemon  | 环境变量 
---|---  
NameNode  | HADOOP_NAMENODE_OPTS  
DataNode  | HADOOP_DATANODE_OPTS  
Secondary NameNode  | HADOOP_SECONDARYNAMENODE_OPTS  
ResourceManager  | YARN_RESOURCEMANAGER_OPTS  
NodeManager  | YARN_NODEMANAGER_OPTS  
WebAppProxy  | YARN_PROXYSERVER_OPTS  
Map Reduce Job History Server  | HADOOP_JOB_HISTORYSERVER_OPTS  
  
例如，为了配置Namenode使用parallelGC，以下表述应该加入到hadoop-env.sh :

    
    
      export HADOOP_NAMENODE_OPTS="-XX:+UseParallelGC"
    

参考`etc/hadoop/hadoop-env.sh` 获取其它例子。

其它重要的配置变量包括：

  * `HADOOP_PID_DIR` \- daemon id 文件的存放路径
  * `HADOOP_LOG_DIR` \- daemon日志文件的存放路径。日志文件如果不存在会在运行时自动创建。
  * `HADOOP_HEAPSIZE` / `YARN_HEAPSIZE` \- 最大使用heapsize值，单位MB，例如该变量设置为1000，则heap被设置为1000MB。用于daemon的heap设置。默认值为1000，可以每个deamon单独配置。

通过情况下，应该将`HADOOP_PID_DIR` 和`HADOOP_LOG_DIR`设置成只有启动hadoop的有写权限的路径，否则容易遭受链接攻击

It is also traditional to configure `HADOOP_PREFIX` in the system-wide shell
environment configuration. For example, a simple script inside
`/etc/profile.d`:

    
    
      HADOOP_PREFIX=/path/to/hadoop
      export HADOOP_PREFIX
    

Daemon  | Environment Variable  
---|---  
ResourceManager  | YARN_RESOURCEMANAGER_HEAPSIZE  
NodeManager  | YARN_NODEMANAGER_HEAPSIZE  
WebAppProxy  | YARN_PROXYSERVER_HEAPSIZE  
Map Reduce Job History Server  | HADOOP_JOB_HISTORYSERVER_HEAPSIZE  
  
### 配置Hadoop Daemons

这一小节描述了下述文件的重要变量：
  * `etc/hadoop/core-site.xml`

变量名| 值| 备注
---|---|---  
`fs.defaultFS` | NameNode URI  | hdfs://host:port/
`io.file.buffer.size` | 131072  | Size of read/write buffer used in SequenceFiles.  
  
  * `etc/hadoop/hdfs-site.xml`

  * 配置NameNode:

变量名| 值| 备注
---|---|---  
`dfs.namenode.name.dir` | Path on the local filesystem where the NameNode stores the namespace and transactions logs persistently.  | If this is a comma-delimited list of directories then the name table is replicated in all of the directories, for redundancy. 
`dfs.hosts` / `dfs.hosts.exclude` | List of permitted/excluded DataNodes.  |If necessary, use these files to control the list of allowable datanodes.  
`dfs.blocksize` | 268435456  | HDFS blocksize of 256MB for large file-systems.  
`dfs.namenode.handler.count` | 100  | More NameNode server threads to handle RPCs from large number of DataNodes.  
  
  * 配置DataNode:

变量名| 值| 备注
---|---|---  
`dfs.datanode.data.dir` | Comma separated list of paths on the local filesystem of a `DataNode` where it should store its blocks.  | If his is a comma-delimited list of directories, then data will be stored in all named directories, typically on different devices.  
  
  * `etc/hadoop/yarn-site.xml`

  * 配置ResourceManager和NodeManager:

变量名| 值| 备注
---|---|---  
`yarn.acl.enable` | `true` / `false` | Enable ACLs? Defaults to _false_.  
`yarn.admin.acl` | Admin ACL  | ACL to set admins on the cluster. ACLs are of for _comma-separated-usersspacecomma-separated-groups_. Defaults to special value of ***** which means _anyone_. Special value of just _space_ means no one has access.  
`yarn.log-aggregation-enable` | _false_ | Configuration to enable or disable log aggregation  
  
  * 配置ResourceManager:

变量名| 值| 备注
---|---|---  
`yarn.resourcemanager.address` | `ResourceManager` host:port for clients to submit jobs.  | _host:port_ If set, overrides the hostname set in `yarn.resourcemanager.hostname`.  
`yarn.resourcemanager.scheduler.address` | `ResourceManager` host:port for ApplicationMasters to talk to Scheduler to obtain resources.  | _host:port_ If set, overrides the hostname set in `yarn.resourcemanager.hostname`.  
`yarn.resourcemanager.resource-tracker.address` | `ResourceManager` host:port for NodeManagers.  | _host:port_ If set, overrides the hostname set in `yarn.resourcemanager.hostname`.  
`yarn.resourcemanager.admin.address` | `ResourceManager` host:port for administrative commands.  | _host:port_ If set, overrides the hostname set in `yarn.resourcemanager.hostname`.
`yarn.resourcemanager.webapp.address` | `ResourceManager` web-ui host:port.  |_host:port_ If set, overrides the hostname set in `yarn.resourcemanager.hostname`.
`yarn.resourcemanager.hostname` | `ResourceManager` host.  | _host_ Single hostname that can be set in place of setting all `yarn.resourcemanager*address` resources. Results in default ports for ResourceManager components.  
`yarn.resourcemanager.scheduler.class` | `ResourceManager` Scheduler class.  |`CapacityScheduler` (recommended), `FairScheduler` (also recommended), or `FifoScheduler`  
`yarn.scheduler.minimum-allocation-mb` | Minimum limit of memory to allocate to each container request at the `Resource Manager`.  | In MBs  
`yarn.scheduler.maximum-allocation-mb` | Maximum limit of memory to allocate to each container request at the `Resource Manager`.  | In MBs 
`yarn.resourcemanager.nodes.include-path` / `yarn.resourcemanager.nodes.exclude-path` | List of permitted/excluded NodeManagers.  | If necessary, use these files to control the list of allowable NodeManagers.    
  * 配置NodeManager:

变量名| 值| 备注 
---|---|---  
`yarn.nodemanager.resource.memory-mb` | Resource i.e. available physical memory, in MB, for given `NodeManager` | Defines total available resources on the `NodeManager` to be made available to running containers  
`yarn.nodemanager.vmem-pmem-ratio` | Maximum ratio by which virtual memory usage of tasks may exceed physical memory  | The virtual memory usage of each task may exceed its physical memory limit by this ratio. The total amount of virtual memory used by tasks on the NodeManager may exceed its physical memory usage by this ratio.  
`yarn.nodemanager.local-dirs` | Comma-separated list of paths on the local filesystem where intermediate data is written.  | Multiple paths help spread disk i/o.  
`yarn.nodemanager.log-dirs` | Comma-separated list of paths on the local filesystem where logs are written.  | Multiple paths help spread disk i/o.  
`yarn.nodemanager.log.retain-seconds` | _10800_ | Default time (in seconds) to retain log files on the NodeManager Only applicable if log-aggregation is disabled.  
`yarn.nodemanager.remote-app-log-dir` | _/logs_ | HDFS directory where the application logs are moved on application completion. Need to set appropriate permissions. Only applicable if log-aggregation is enabled.  
`yarn.nodemanager.remote-app-log-dir-suffix` | _logs_ | Suffix appended to the remote log dir. Logs will be aggregated to ${yarn.nodemanager.remote-app-log-dir}/${user}/${thisParam} Only applicable if log-aggregation is enabled.  
`yarn.nodemanager.aux-services` | mapreduce_shuffle  | Shuffle service that needs to be set for Map Reduce applications.  
  
  * 配置History Server (需要移到别处):

变量名| 值| 备注
---|---|---  
`yarn.log-aggregation.retain-seconds` | _-1_ | How long to keep aggregation logs before deleting them. -1 disables. Be careful, set this too small and you will spam the name node.  
`yarn.log-aggregation.retain-check-interval-seconds` | _-1_ | Time between checks for aggregated log retention. If set to 0 or a negative value then the value is computed as one-tenth of the aggregated log retention time. Be careful, set this too small and you will spam the name node.  
  
  * `etc/hadoop/mapred-site.xml`

  * 配置MapReduce应用:

变量名| 值| 备注
---|---|---  
`mapreduce.framework.name` | yarn  | Execution framework set to Hadoop YARN.  
`mapreduce.map.memory.mb` | 1536  | Larger resource limit for maps.  
`mapreduce.map.java.opts` | -Xmx1024M  | Larger heap-size for child jvms of maps.  
`mapreduce.reduce.memory.mb` | 3072  | Larger resource limit for reduces.  
`mapreduce.reduce.java.opts` | -Xmx2560M  | Larger heap-size for child jvms of reduces.  
`mapreduce.task.io.sort.mb` | 512  | Higher memory-limit while sorting data for efficiency.  
`mapreduce.task.io.sort.factor` | 100  | More streams merged at once while sorting files.  
`mapreduce.reduce.shuffle.parallelcopies` | 50  | Higher number of parallel copies run by reduces to fetch outputs from very large number of maps.  
  
  * 配置MapReduce JobHistory Server:

变量名| 值| 备注
---|---|---  
`mapreduce.jobhistory.address` | MapReduce JobHistory Server _host:port_ |Default port is 10020.  
`mapreduce.jobhistory.webapp.address` | MapReduce JobHistory Server Web UI_host:port_ | Default port is 19888.  
`mapreduce.jobhistory.intermediate-done-dir` | /mr-history/tmp  | Directory where history files are written by MapReduce jobs.  
`mapreduce.jobhistory.done-dir` | /mr-history/done  | Directory where history files are managed by the MR JobHistory Server.  
  
## 监控NodeManagers健康状态

Hadoop提供一个机制，管理员可以通过配置NodeManager运行一个定期提供一个节点是否健康的脚本。

管理员通过脚本检测节点的健康状态。如果脚本检测节点不健康，必须以ERROR开头打印到标准输出文件。NodeManager大量产生定期脚本并检测它们的输出。如果脚本的输出包含ERROR，如上所述，节点的状态被报告为`unhealthy`，节点被ResourceManager列为黑名单中，不再有更多的任务派发到该节点。然后，NodeManager继续运行脚本，如果该节点恢复健康，ResourceManager将它被从黑名单中自动移除。节点的健康状态均在脚本的输出中，如果不健康，管理员在ResourceManager的web界面中可以查看。不健康的时长也可以web界面中展示。

在`etc/hadoop/yarn-site.xml`中，以下变量值可以用来控制监控节点的健康程度


变量名| 值| 备注
---|---|---  
`yarn.nodemanager.health-checker.script.path` | Node health script  | Script to check for node's health status.  
`yarn.nodemanager.health-checker.script.opts` | Node health script options  |Options for script to check for node's health status.  
`yarn.nodemanager.health-checker.script.interval-ms` | Node health script interval  | Time interval for running health script.  
`yarn.nodemanager.health-checker.script.timeout-ms` | Node health script timeout interval  | Timeout for health script execution.  
  

如果只是本地磁盘出现故障，健康检测脚本不支持提示ERROR。NodeManager有能力定期检测磁盘的健康度（尤其是nodemanager-local-dirs和nodemanager-log-dirs），在坏的目录数量达到阈值（由yarn.nodemanager.disk-health-checker.min-healthy-disks决定）时，节点会被标记为不健康并将该信息发送到资源管理器resource manager。一个可能性是磁盘被搜查了，另外一种可能是健康监测脚本查出了一个故障。

## Slaves文件

在`etc/hadoop/slaves`文件写出所有slave的hostname或者IP地址，每一个一行。Helper scripts（如下所述）使用`etc/hadoop/slaves`文件在各服务器上立刻执行脚本。它并不使用任何基于Java的Hadoop配置。为了正常使用，运行Hadoop的用户各服务器之间需要ssh免认证（服务器之间ssh免认证，或其它手段，例如Kerberos）

## Hadoop机架感知

许多Hadoop组成是机架感知的，通过网络拓扑达到性能和安全性的优势。Hadoop进程通过管理员配置模块获取集群中slave的信息。参考[Rack Awareness](http://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-common/RackAwareness.html) 获取更多信息

强烈推荐在启动HDFS之前配置机架感知。

## 日志

Hadoop在Apache通用日志框架下使用[Apache log4j](http://logging.apache.org/log4j/2.x/)记录日志。编辑`etc/hadoop/log4j.properties`更改配置（例如日志格式等）

## 操作Hadoop集群

一旦完成所有必须的配置，将所有的配置文件放到`HADOOP_CONF_DIR`目录下，这个目录在所有服务器应该相同。

通常来说，推荐HDFS和YARN使用单独的用户运行。大多数的安装中，HDFS进程使用hdfs用户，YARN使用yarn用户

### 启动Hadoop

启动Hadoop集群需要将HDFS和YARN都启动。

第一次启动HDFS，必须格式化。格式化成为一个分布式文件系统_hdfs_:

    
    
    [hdfs]$ $HADOOP_PREFIX/bin/hdfs namenode -format <cluster_name>
    

在指定节点使用以下命令将HDFS NameNode启动为_hdfs_:


    
    
    [hdfs]$ $HADOOP_PREFIX/sbin/hadoop-daemon.sh --config $HADOOP_CONF_DIR --script hdfs start namenode
    

在指定节点使用以下命令将HDFS DataNode启动为_hdfs_: 

    
    
    [hdfs]$ $HADOOP_PREFIX/sbin/hadoop-daemons.sh --config $HADOOP_CONF_DIR --script hdfs start datanode
    

如果`etc/hadoop/slaves`和ssh免认证已经配置好（参见 [Single Node Setup](http://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-common/SingleCluster.html)），所有的HDFS进程可以通过以下脚本启动。作为_hdfs_:

    
    
    [hdfs]$ $HADOOP_PREFIX/sbin/start-dfs.sh
    

使用下列命令启动YARN，在指定节点上作为_yarn_运行：


    
    
    [yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR start resourcemanager
    

使用脚本在指定host上启动NodeManager用作_yarn_运行：

    
    
    [yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemons.sh --config $HADOOP_CONF_DIR start nodemanager
    

启动一个单独的WebAppProxy服务器，用作_yarn_运行。如果多台服务器使用负载均衡，该服务将在它们每一台运行：

    
    
    [yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR start proxyserver
    

如果`etc/hadoop/slaves`和ssh免认证已经配置好（参见 [Single Node Setup](http://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-common/SingleCluster.html)），所有的YARN进程可以通过以下脚本启动。作为_yarn_:


    
    
    [yarn]$ $HADOOP_PREFIX/sbin/start-yarn.sh
    


使用下列命令启动MapReduce JobHistory服务，用作_mapred_:

    
    
    [mapred]$ $HADOOP_PREFIX/sbin/mr-jobhistory-daemon.sh --config $HADOOP_CONF_DIR start historyserver
    

### 关闭Hadoop

使用下列命令关闭用作_hdfs_的NameNode：


    
    
    [hdfs]$ $HADOOP_PREFIX/sbin/hadoop-daemon.sh --config $HADOOP_CONF_DIR --script hdfs stop namenode
    

使用脚本停止_hdfs_的DataNode：

    
    
    [hdfs]$ $HADOOP_PREFIX/sbin/hadoop-daemons.sh --config $HADOOP_CONF_DIR --script hdfs stop datanode
    

如果`etc/hadoop/slaves`和ssh免认证已经配置好（参见 [Single Node Setup](http://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-common/SingleCluster.html)），所有的HDFS进程可以通过以下脚本停止。作为_hdfs_:

    
    
    [hdfs]$ $HADOOP_PREFIX/sbin/stop-dfs.sh
    

使用下列命令关闭指定节点用作_yarn_的ResourceManager:


    
    
    [yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR stop resourcemanager
    

使用_yarn_脚本在slave停止NodeManager: 

    
    
    [yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemons.sh --config $HADOOP_CONF_DIR stop nodemanager
    

如果`etc/hadoop/slaves`和ssh免认证已经配置好（参见 [Single Node Setup](http://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-common/SingleCluster.html)），所有的YARN进程可以通过_yarn_脚本停止:

    
    
    [yarn]$ $HADOOP_PREFIX/sbin/stop-yarn.sh
    

停止WebAppProxy服务，通过_yarn_运行WebAppProxy ，如果多台服务器使用负载均衡，以下命令要在每一台运行：

    
    
    [yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR stop proxyserver
    

使用下述命令停止MapReduce JobHistory服务，在指定节点上用作_mapred_运行：

    
    
    [mapred]$ $HADOOP_PREFIX/sbin/mr-jobhistory-daemon.sh --config $HADOOP_CONF_DIR stop historyserver
    

## Web接口

Hadoop集群搭建成功后可以通过下表检验web接口


Daemon  | Web 接口| 备注
---|---|---  
NameNode  | <http://nn_host:port/> | Default HTTP port is 50070.  
ResourceManager  | <http://rm_host:port/> | Default HTTP port is 8088.  
MapReduce JobHistory Server  | <http://jhs_host:port/> | Default HTTP port is
19888.
