#HDFS Users Guide

----------


##purpose

本文档是HDFS(Hadoop Distributed File System)用户的基础教程(starting point)。HDFS是hadoop单机或者集群的分布式文件系统，在许多生产环境下被设计成可行的。具有HDFS实际经验能极大帮助改进配置和在具体集群环境下的诊断。

##Overview

HDFS是Hadoop应用的一个最主要的分布式文件存储系统。一个HDFS集群主要有一个Namenode和多个Datanode组成：Namenode管理文件系统的元数据，而Datanode存储了实际的数据。[HDFS体系结构指南](http://hadoop.apache.org/docs/r2.6.4/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)中有详细的描述。
本文档主要关注HDFS集群与用户和管理员的交互。在HDFS体系结构图中描述了NameNode、DataNode和客户端之间的基本操作。客户端向NameNode请求文件元数据或者文件修饰属性，而真正的文件I/O操作是直接和DataNode交互的。
下面列出了一些多数用户都比较感兴趣的重要特性。

 * hadoop(包括HDFS)非常适合商用的分布式存储和分布式计算，因为它不仅具有容错性和可扩展性，而且非常易于扩展。MapReduce框架以其在大型分布式应用中的简单性和可用性而著称，已经被集成到hadoop中。
 * HDFS的可配置性极高，同时它的默认配置能够满足很多安装环境。多数情况下，这些参数只在非常大规模的机器中才需要调整。
 * 用java开发，支持所有的主流平台。
 * 支持类shell命令，可直接与HDFS交互。
 * NameNode和DataNode有内置的Web服务器，方便用户检查集群的当前状态。
 * 新特性和改进会定期加入HDFS的实现中。下面列出的是HDFS中常用特性的一部分：

1. 文件权限和授权。

2. 机架感知(Rack awareness):在调度任务和分配存储空间时考虑了节点的物理节点。

3. 安全模式：一种需要维护的管理模式。

4. fsck：一个诊断文件系统健康状况的工具，能够发现丢失的文件或数据块。

5. Balancer：当datanode之间数据不均衡时，平衡集群上的数据负载。

6. 升级和回滚：在软件更新后有异常发生的情形下，能够回滚到HDFS升级之前的状态。

7. Secondary Namenode：对文件系统名字空间执行周期性的检查点，将Namenode上HDFS改动日志文件的大小控制在某个特定的限度下。

8. Checkpoint node: 定期的检查命名空间以助于缩小在Namenode上的包含HDFS变动的日志文件的大小。

9. Backup node： checkpoint的扩展. 除了checkpointing其还可以接受Namenode来的可编辑的数据流并且在内存中维护一份属于自己的namespace, 并且和激活的Namesapce的状态保持同步，只有Backup节点可以和namespace同时注册。

## 必备条件

下面的文档描述了如何安装和搭建Hadoop集群：

* [Single Node Setup](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html)
* [Cluster Setup ](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/ClusterSetup.html)

文档的剩余部分假设用户已经安装并运行了至少包含一个DataNode的HDFS。就本文目的来说，NameNode和DataNode可以运行在同一个物理机器上。

## Web接口

NameNode和DataNode每个都在内部运行了WEB服务，用来显示集群的现状的基本信息。在默认配置下，NameNode的首页是：http://namenode-name:50070，它列出了在集群中的数据节点和集群的基本情况。web接口也可以用来浏览文件系统(在NameNode首页上使用 “Browse the file system” 链接)。

## shell命令

Hadoop包含一系列的类shell的命令，可直接和HDFS以及其它Hadoop支持的文件系统交互。

>bin/hadoop fs -help

命令列出所有Hadoop Shell支持的命令。而 

>bin/hadoop fs -help command-name 

命令能显示关于某个命令的详细信息。这些命令支持大多数普通文件系统的操作，比如复制文件、改变文件权限等。它还支持一些HDFS特有的操作，比如改变文件副本数目。
更多信息参考[File System Shell Guide](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/FileSystemShell.html)

### DFSAdmin 命令

>bin/hdfs dfsadmin

该命令支持一些管理员相关的操作。

> bin/hdfs dfsadmin -help

列出了当前所有支持的命令。例如：

* -report： 打印HDFS基本的统计信息，一些信息可以在NameNode的首页上看到。
* -safemode：虽然通常并不需要，管理员可以手动进入或者离开安全模式。
* -refreshNode:在NameNode上更新那些允许连接到namenode的datanodes。Namenodes重新读取到在df.hosts中定义的dfs.hosts,dfs.hosts.exclude中的datanode主机名。如果配置了dfs.hosts，那么只有其中的主机允许注册到namenode。配置在dfs.hosts.exclude的那些datanodes是需要被剔除掉的。当datanode上的副本完全复制到其他datanode上时，节点才完全的被剔除。被剔除的节点不会自动的关闭，并且不会选择写入新的副本。
* -printTopology： 打印集群拓扑。显示一个树形构架，datanode会作为Namenode的一个视图添加到构架中。

## Secondary NameNode

NameNode将对文件系统的改动追加保存到本地文件系统上的一个日志文件(edits)。当一个NameNode启动后，它从一个镜像文件(fsimage)中读取HDFS的状态,接着应用日志文件中的edits操作。然后它将新的HDFS状态写入(fsimage)中，并使用一个空的edits文件开始正常操作。因为NameNode只有在启动阶段才合并fsimage和edits，所以久而久之日志文件可能会变得非常庞大，特别是对大型的集群。日志文件太大的另一个副作用是下一次NameNode启动会花很长的时间。

Secondary NameNode定期合并fsimage和edits日志，并将edits日志文件大小控制在一个限度下。因为内存它的内存需求和NameNode在一个数量级上，所以通常secondary NameNode和NameNode运行在不同的机器上。

secondary NameNode的checkpoint进程启动，是由两个参数控制的：

* fs.checkpoint.period，指定连续两次检查点的最大时间间隔， 默认值是1小时。
* fs.checkpoint.txns定义了在NameNode上没有被checkpoint的事务的数量，一旦超过这个值这个NameNode会强制checkpoint，即使没有到checkpoint的时间间隔1000000。

Secondary NameNode保存最新检查点的目录与NameNode的目录结构相同。 所以NameNode可以在需要的时候读取Secondary NameNode上的检查点镜像。

## Checkpoint Node

NameNode使用两个文件来持久化它的命名空间(namespace)：fsimage和edits。
fsimage是namespace的最近的一个checkpoint的一个镜像。
edits是从checkpoint开始保存namespace的变化的一个日志。
当NameNode启动时，它合并fsimage和edits来提供文件系统的最新信息。
然后NameNode会用HDFS最新的状态来覆盖fsimage(还是一个fsimage)并生成一个新的edits日志文件。

Checkpoint node定期创建namespace的checkpoints。它从存活的NameNode上下载fsimage和edits，并在本地合并他们，并且上传新的image到存活的NameNode。因为内存和NameNode是一个数量级别，所以Checkpoint node一般和NameNode运行在不同的机器上。
Checkpoint node一般由配置文件指定的节点上启动。

> bin/hdfs namenode -checkpoint

Checkpoint(or backup) 节点的位置和相应的web接口是用通过

> dfs.namenode.backup.address 
> 
> dfs.namenode.backup.http-address

来指定的。

在Checkpoint节点上的Checkpoint进程的启动是由下面两个参数控制的：

* dfs.namenode.checkpoint.period 指定连续两次检查点的最大时间间隔， 默认值是1小时。
* dfs.namenode.checkpoint.txns 定义了在NameNode上没有被checkpoint的事务的数量，一旦超过这个值这个NameNode会强制checkpoint，即使没有到checkpoint的时间间隔1000000。

Checkpoint节点保存最新检查点的目录与NameNode的目录结构相同。 所以NameNode可以在需要的时候读取Secondary NameNode上的检查点镜像。

## Backup Node

备份节点提供了和检查点一样的功能，同时在内存中维护一个从活动的NameNode拷贝的副本，总是同步最新的文件系统副本。随着从NameNode接收文件系统的日志流并且保存到磁盘，备份节点同样应用这些edits到自己的内存里的命名空间的副本中，这样就创建了一个命名空间的备份。

备份节点不需要为了创建一个Checkpoint，而从活动的NameNode上下载fsimage和edits文件，作为一个检查点节点和secondary NameNode是需要的，因为备份节点以及拥有了内存中的命名空间的最新状态。备份节点的Checkpoint node的进程因为只需要保存命名空间到本地fsimage和重置edits，所以更高效。

作为备份节点要在内存中维护一份命名空间的拷贝，它们的RAM(内存大小)需要和NameNode节点一样。

NameNode在同一时间只支持一个备份节点。如果使用了一个备份节点，那么将不能注册检查点节点。同时支持多个备份节点将在将来支持。

备份节点和检查点节点的配置一样。它使用
> bin/hdfs namenode -backup

命令来启动。

备份（或者检查点）节点，它们伴随的web接口的位置是通过,dfs.namenode.backup.address和dfs.name.backup.http-address的值来配置。

使用一个备份节点提供了选项在运行NameNode时，将非持久性存储，所有命名空间状态持久性的责任，抛给了备份节点。为了做这些，开启NameNode使用-importCheckPoint选项，同时在NameNode中，为edits类型设置非持久性存储目录 dfs.namenode.edits.dir

## Import Checkpoint

如果所有的image拷贝和edits文件都丢失了，最近的检查点可以被导入到NameNode。
为了做到这一点需要：

* 创建一个在dfs.namenode.name.dir中配置值的空路径；
* 在dfs.namenode.checkpoint.dir配置的值中指定检查点目录的位置；
* 使用-importCheckpoint选项启动NameNode。

NameNode将从dfs.namenode.checkoint.dir路径中上传检查点，并且保存它到dfs.namenode.name.dir中配置的NameNode 目录。如果在dfs.namenode.name.dir中存在一个合法的image，NameNode将失败。 NameNode验证 dfs.namenode.checkpoint.dir中image一致性，但不以任何方式修改它。
更多命令使用，可以参考[namenode](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#namenode)

## balancer

HDFS的数据也许并不是非常均匀的分布在各个DataNode中。一个常见的原因是在现有的集群上经常会增添新的DataNode节点。当新增一个数据块（一个文件的数据被保存在一系列的块中）时，NameNode在选择DataNode接收这个数据块之前，会考虑到很多因素。其中的一些考虑的是：

* 将数据块的一个副本放在正在写这个数据块的节点上。
* 尽量将数据块的不同副本分布在不同的机架上，这样集群可在完全失去某一机架的情况下还能存活。
* 一个副本通常被放置在和写文件的节点同一机架的某个节点上，这样可以减少跨越机架的网络I/O。
* 尽量均匀地将HDFS数据分布在集群的DataNode中。

由于上述多种考虑需要取舍，数据可能并不会均匀分布在DataNode中。HDFS为管理员提供了一个工具，用于分析数据块分布和重新平衡DataNode上的数据分布。[HADOOP-1652](https://issues.apache.org/jira/browse/HADOOP-1652)是一个简要的指南。

## Rack Awareness

通常，大型Hadoop集群是以机架的形式来组织的，同一个机架上不同节点间的网络状况比不同机架之间的更为理想。另外，NameNode设法将数据块副本保存在不同的机架上以提高容错性。Hadoop允许集群的管理员通过配置dfs.network.script参数来确定节点所处的机架。当这个脚本配置完毕，每个节点都会运行这个脚本来获取它的机架ID。默认的安装假定所有的节点属于同一个机架。[HADOOP-692](https://issues.apache.org/jira/browse/HADOOP-692)有更详细的描述。

## Safemode

NameNode启动时会从fsimage和edits日志文件中装载文件系统的状态信息，接着它等待各个DataNode向它报告它们各自的数据块状态，这样，NameNode就不会过早地开始复制数据块，即使在副本充足的情况下。这个阶段，NameNode处于安全模式下。NameNode的安全模式本质上是HDFS集群的一种只读模式，此时集群不允许任何对文件系统或者数据块修改的操作。通常NameNode会在开始阶段自动地退出安全模式。如果需要，你也可以通过'bin/hadoop dfsadmin -safemode'命令显式地将HDFS置于安全模式。NameNode首页会显示当前是否处于安全模式。关于安全模式的更多介绍和配置信息请参考JavaDoc：setSafeMode()。

## fsck(修复)

HDFS支持fsck命令来检查系统中的各种不一致状况。这个命令被设计来报告各种文件存在的问题，比如文件缺少数据块或者副本数目不够。不同于在本地文件系统上传统的fsck工具，这个命令并不会修正它检测到的错误。一般来说，NameNode会自动修正大多数可恢复的错误。HDFS的fsck不是一个Hadoop shell命令。它通过'bin/hadoop fsck'执行。 命令的使用方法请参考[fsck](http://hadoop.apache.org/docs/r2.6.4/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#fsck),fsck可用来检查整个文件系统，也可以只检查部分文件。

## fetchdt

HDFS通过fetchdt命令来获得Delegation Token，然后存储到一个本地文件系统的文件中。这个token可以在之后从一个非安全客户端来访问安全服务器（例如NameNode）。程序使用RPC或者HTTPS来获得token，因此在运行之前需要存在kerberos tickets（运行kinit来获得tickets）。HDFS fetchdt命令不是一个Hadoop shell命令。可以使用bin/hdfs fetchdt DTfile来运行。在你获得token之后，可以允许一个HDFS命令而没有kerberos tickets，通过制定HADOOP_TOKEN_FILE_LOCATION环境变量来表示token文件。

## recovery mode

通常，你将配置多个元数据存储位置。这样，如果一个存储位置坏了，你可以从其他存储位置来读取元数据。
但是，唯一存储位置损坏了，你能够做些什么？在这个场景下，可以指定NameNode启动模式调用Recovery mode，可能会让你恢复你的大部分数据。
你可以在恢复模式开启NameNode，例如这样：namenode -recover
当在恢复模式时，NameNode将会以交互式的提示你在命令行，采取可能的行动来恢复你的数据。
如果你不想被体制，你可以使用 -force选项。这个选项将强制恢复模式总是选择第一个选项。通常情况下，这些都是合理的选择。
因为恢复模式会导致丢失数据，你需要在使用它之前备份你的edit日志和fsimage。

## Upgrade and Rollback

当在一个已有集群上升级Hadoop时，像其他的软件升级一样，可能会有新的bug或一些会影响到现有应用的非兼容性变更出现。在任何有实际意义的HDSF系统上，丢失数据是不被允许的，更不用说重新搭建启动HDFS了。HDFS允许管理员退回到之前的Hadoop版本，并将集群的状态回滚到升级之前。更多关于HDFS升级的细节在升级[wiki](http://wiki.apache.org/hadoop/Hadoop_Upgrade)上可以找到。HDFS在一个时间可以有一个这样的备份。在升级之前，管理员需要用bin/hadoop dfsadmin -finalizeUpgrade（升级终结操作）命令删除存在的备份文件。下面简单介绍一下一般的升级过程：

* 升级 Hadoop 软件之前，请检查是否已经存在一个备份，如果存在，可执行升级终结操作删除这个备份。通过dfsadmin -upgradeProgress status命令能够知道是否需要对一个集群执行升级终结操作。
* 停止集群并部署新版本的Hadoop。
* 使用-upgrade选项运行新的版本（bin/start-dfs.sh -upgrade）。
* 在大多数情况下，集群都能够正常运行。一旦我们认为新的HDFS运行正常（也许经过几天的操作之后），就可以对之执行升级终结操作。注意，在对一个集群执行升级终结操作之前，删除那些升级前就已经存在的文件并不会真正地释放DataNodes上的磁盘空间。
* 如果需要退回到老版本，那停止集群并且部署老版本的Hadoop，然后用回滚选项启动集群（bin/start-dfs.h -rollback）。

## File Permissions and Security

这里的文件权限和其他常见平台如Linux的文件权限类似。目前，安全性仅限于简单的文件权限。启动NameNode的用户被视为HDFS的超级用户。HDFS以后的版本将会支持网络验证协议（比如Kerberos）来对用户身份进行验证和对数据进行加密传输。

## Scalability

现在，Hadoop已经运行在上千个节点的集群上。[PoweredBy wiki](http://wiki.apache.org/hadoop/PoweredBy)页面列出了一些已将Hadoop部署在他们的大型集群上的组织。HDFS集群只有一个NameNode节点。目前，NameNode上可用内存大小是一个主要的扩展限制。在超大型的集群中，增大HDFS存储文件的平均大小能够增大集群的规模，而不需要增加NameNode的内存。默认配置也许并不适合超大规模的集群。[Hadoop FAQ](http://wiki.apache.org/hadoop/FAQ)页面列举了针对大型Hadoop集群的配置改进。

## Related Documentation

这个用户手册给用户提供了一个学习和使用HDSF文件系统的起点。本文档会不断地进行改进，同时，用户也可以参考更多的Hadoop和HDFS文档。下面的列表是用户继续学习的起点：

* [hadoop官方主页](http://hadoop.apache.org/)
* [hadoop wiki](http://wiki.apache.org/hadoop/FrontPage)：Hadoop Wiki文档首页。这个指南是Hadoop代码树中的一部分，与此不同，Hadoop Wiki是由Hadoop社区定期编辑的。
* [hadoop FAQ](http://wiki.apache.org/hadoop/FAQ)
* [hadoop java api](http://hadoop.apache.org/docs/r2.6.4/api/index.html)
* Hadoop User Mailing List: user[at]hadoop.apache.org.
* 查看conf/hadoop-default.xml文件。这里包括了大多数配置参数的简要描述。
* [HDFS Commands Guide](http://hadoop.apache.org/docs/r2.6.4/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html)