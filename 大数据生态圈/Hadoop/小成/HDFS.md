# 分布式文件系统——HDFS


## 一、基本概念
**HDFS**（**Hadoop Distributed File System**）是一个易于扩展的分布式文件系统，运行在成百上千低成本的机器上。具有高容错、高吞吐量等特性，主要用于对海量文件信息进行存储和管理。


## 二、系统架构

HDFS采用了主从（Master/Slave）结构模型，一个HDFS集群是由一个NameNode和若干个DataNode组成的。其中NameNode作为主服务器，管理文件系统的命名空间和客户端对文件的访问操作；集群中的DataNode管理存储的数据。

### HDFS构成组件及作用：
- **NameNode**：

&nbsp;&nbsp;管理着文件系统命名空间
&nbsp;&nbsp;&nbsp;&nbsp;维护着文件系统树及树中的所有文件和目录

&nbsp;&nbsp;存储元数据
&nbsp;&nbsp;&nbsp;&nbsp;NameNode保存元信息的种类有：
&nbsp;&nbsp;&nbsp;&nbsp;文件名目录名及它们之间的层级关系
&nbsp;&nbsp;&nbsp;&nbsp;文件目录的所有者及其权限
&nbsp;&nbsp;&nbsp;&nbsp;每个文件块的名及文件有哪些块组成

&nbsp;&nbsp;元数据保存在内存中
&nbsp;&nbsp;&nbsp;&nbsp;NameNode元信息并不包含每个块的位置信息

&nbsp;&nbsp;保存文件，block，datanode之间的映射关系：文件名到block，block到datanode

&nbsp;&nbsp;元信息持久化
&nbsp;&nbsp;&nbsp;&nbsp;在NameNode中存放元信息的文件是 fsimage。在系统运行期间所有对元信息的操作都保存在内存中并被持久化到另一个文件edits中。并且edits文件和fsimage文件会被SecondaryNameNode周期性的合并

&nbsp;&nbsp;运行NameNode会占用大量内存和I/O资源，一般NameNode不会存储用户数据或执行MapReduce任务。

&nbsp;&nbsp;HDFS更倾向存储大文件原因：
&nbsp;&nbsp;&nbsp;&nbsp;一般来说，一条元信息记录会占用200byte内存空间。假设块大小为64MB，备份数量是3 ，那么一个1GB大小的文件将占用16\*3=48个文件块。如果现在有1000个1MB大小的文件，则会占用1000\*3=3000个文件块（多个文件不能放到一个块中）。我们可以发现，如果文件越小，存储同等大小文件所需要的元信息就越多，所以HDFS更喜欢大文件。

&nbsp;&nbsp;Hadoop系统只有一个NameNode
&nbsp;&nbsp;&nbsp;&nbsp;导致单点问题
&nbsp;&nbsp;&nbsp;&nbsp;两种解决方案：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;将hadoop元数据写入到本地文件系统的同时再实时同步到一个远程 挂载的网络文件系统（NFS）。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;运行一个secondary NameNode，它的作用是与NameNode进行交 互，定期通过编辑日志文件合并命名空间镜像，当NameNode发生故 障时它会通过自己合并的命名空间镜像副本来恢复。需要注意的是 secondaryNameNode保存的状态总是滞后于NameNode，所以这 种方式难免会导致丢失部分数据。

- **SecondaryNameNode**:

&nbsp;&nbsp;命名不好，容易误解，SecondNameNode并不是NameNode的备份。
&nbsp;&nbsp;用来保存HDFS的元数据信息，比如命名空间信息、块信息等，由于这些信息是在内存的，SecondNameNode是为了考虑持久化到磁盘。
&nbsp;&nbsp;作用是与NameNode进行交互，定期通过编辑日志（edit log）文件合并命名空间镜像(fsimage)，当NameNode发生故障时它会通过自己合并的命名空间镜像副本来恢复。
&nbsp;&nbsp;需要注意的是 secondaryNameNode保存的状态总是滞后于NameNode，所以这种方式难免会导致丢失部分数据。
&nbsp;&nbsp;Secondary NameNode所做的不过是在文件系统 中设置一个检查点来帮助NameNode更好的工作。 它不是要取代掉NameNode也不是NameNode的备份。
定时到NameNode去获取edit logs，并更新到fsimage[Secondary NameNode自己的fsimage]。一旦它有了新的fsimage文件，它将其拷贝回NameNode中。NameNode在下次重启时会使用这个新的fsimage文件，从而减少重启的时间。

- **DataNode**：

&nbsp;&nbsp;作用：
&nbsp;&nbsp;&nbsp;&nbsp;负责存储数据块，负责为系统客户端提供数据块的读写服务
&nbsp;&nbsp;&nbsp;&nbsp;根据NameNode的指示进行创建、删除和复制等操作
&nbsp;&nbsp;&nbsp;&nbsp;心跳机制，定期报告文件块列表信息
&nbsp;&nbsp;&nbsp;&nbsp;DataNode之间进行通信，块的副本处理

&nbsp;&nbsp;数据块——磁盘读写的基本单位
&nbsp;&nbsp;&nbsp;&nbsp;HDFS默认数据块大小64MB或128M
&nbsp;&nbsp;&nbsp;&nbsp;磁盘块一般为512B
&nbsp;&nbsp;&nbsp;&nbsp;原因：块增大可以减少寻址时间，降低寻址时间/文件传输时间，若寻址时间为10ms，磁盘传输速率为100MB/s，那么该比例仅为1%，数据块过大也不好，因为一个MapReduce通常以一个块作为输入，块过大会导致整体任务数量过小，降低作业处理速度

&nbsp;&nbsp;机架感知策略 —— Block副本放置策略
&nbsp;&nbsp;&nbsp;&nbsp;第一个副本，在客户端相同的节点（如果客户端是集群外的一台机器，就随机算节点，但是系统会避免挑选太满或者太忙的节点）
&nbsp;&nbsp;&nbsp;&nbsp;第二个副本，放在不同机架（随机选择）的节点
&nbsp;&nbsp;&nbsp;&nbsp;第三个副本，放在与第二个副本同机架但是不同节点上
&nbsp;&nbsp;&nbsp;&nbsp;距离是标准化的，不可修改
&nbsp;&nbsp;&nbsp;&nbsp;distance(/D1/R1/H1,/D1/R1/H1)=0 相同的datanode
&nbsp;&nbsp;&nbsp;&nbsp;distance(/D1/R1/H1,/D1/R1/H2)=2 同一rack下的不同datanode
&nbsp;&nbsp;&nbsp;&nbsp;distance(/D1/R1/H1,/D1/R1/H4)=4 同一IDC下的不同datanode
&nbsp;&nbsp;&nbsp;&nbsp;distance(/D1/R1/H1,/D2/R3/H7)=6 不同IDC下的datanode
&nbsp;&nbsp;&nbsp;&nbsp;这些0,2,4,6是固定的

&nbsp;&nbsp;数据完整性校验
&nbsp;&nbsp;&nbsp;&nbsp;不希望在存储和处理数据时丢失或损坏任何数据
&nbsp;&nbsp;&nbsp;&nbsp;HDFS 会对写入的数据计算校验和，并在读取数据时验证校验和
&nbsp;&nbsp;&nbsp;&nbsp;两种检验方法：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;校验和：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;import binascii
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;binascii.crc32
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;检测损坏数据的常用方法是在第一次进行系统时计算数据的校验和，在通道传输过程中，如果新生成的校验和不完全匹配原始的校验和，那么数据就会被认为是被损坏的。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;数据块检测程序DataBlockScanner
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在DataNode节点上开启一个后台线程，来定期验证存储在它上所有块，这个是防止物理介质出现损减情况而造成的数据损坏。

&nbsp;&nbsp;容错 - 可靠性措施
&nbsp;&nbsp;&nbsp;&nbsp;一个名字节点和多个数据节点
&nbsp;&nbsp;&nbsp;&nbsp;数据复制（冗余机制）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;存放的位置（机架感知策略）
&nbsp;&nbsp;&nbsp;&nbsp;故障检测
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;数据节点
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;心跳包（检测是否宕机）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;快报告（安全模式下检测）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;数据完整性检测（校验和比较）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;名字节点（日志文件，镜像文件）

&nbsp;&nbsp;空间回收机制
&nbsp;&nbsp;&nbsp;&nbsp;Trash目录
&nbsp;&nbsp;&nbsp;&nbsp;<property>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<name>fs.trash.interval</name>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<value>1440</value>
&nbsp;&nbsp;&nbsp;&nbsp;</property>
&nbsp;&nbsp;&nbsp;&nbsp;<property>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<name>fs.trash.checkpoint.interval</name>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<value>1440</value>
&nbsp;&nbsp;&nbsp;&nbsp;</property>

&nbsp;&nbsp;fs.trash.interval是在指在这个回收周期之内，文件实际上是被移动到trash的这个目录下面，而不是马上把数据删除掉。等到回收周期真正到了以后，hdfs才会将数据真正删除。默认的单位是分钟，1440分钟=60\*24，刚好是一天。 
fs.trash.checkpoint.interval则是指垃圾回收的检查间隔，应该是小于或者等于fs.trash.interval。


## 三、HDFS 特 点

能做什么
&nbsp;&nbsp;存储并管理PB级数据
&nbsp;&nbsp;处理非结构化数据
&nbsp;&nbsp;注重数据处理的吞吐量（延迟不敏感）
&nbsp;&nbsp;应用模式：write-once-read-many存取模式（无数据一致性问题）

不适合做
&nbsp;&nbsp;存储小文件（不建议）
&nbsp;&nbsp;大量随机读（不建议）
&nbsp;&nbsp;需要对文件修改（不支持）
&nbsp;&nbsp;多用户写入（不支持）
