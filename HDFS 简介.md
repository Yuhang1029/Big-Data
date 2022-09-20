# HDFS 简介

**参考文档**

[Hadoop 大数据概述](https://www.w3cschool.cn/hadoop/hadoop_big_data_overview.html)

[【HDFS】一、HDFS简介及基本概念](https://www.cnblogs.com/gzshan/p/10981007.html)

[Hadoop之HDFS简介](https://www.infoq.cn/article/rudfbprmb5vwfpydus5r)

[Apache Hadoop 3.3.4 - HDFS Architecture](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)

---

## 概述

Hadoop 文件系统 (Hadoop Distributed File System) 是使用分布式文件系统设计开发的，它运行在商用硬件上。与其他分布式系统不同，HDFS 是高度容错的，并且使用低成本硬件设计。HDFS 拥有大量的数据并提供更容易的访问。为了存储这样巨大的数据，文件存储在多个机器。这些文件以冗余方式存储，以在发生故障时避免系统可能的数据丢失。HDFS 还使应用程序可用于并行处理。HDFS 有以下几个显著的特点：

* **可靠性**。HDFS 集群的设备不需要多么昂贵和特殊，只要是一些日常使用的普通硬件即可，正因为如此，HDFS 节点故障的可能性还是很高的，所以必须要有机制来处理这种单点故障，保证数据的可靠。因此错误检测和快速、自动的恢复是 HDFS 最核心的架构目标。

* **超大文件**。目前的 Hadoop 集群能够存储几百 TB 甚至 PB 级的数据。

* **流式数据访问**。HDFS 的访问模式是：**一次写入，多次读取**，更加关注的是读取整个数据集的整体时间，而不是用户交互处理。比之数据访问的低延迟问题，更关键的在于数据访问的高吞吐量。

* **单用户写入，不支持任意修改**。HDFS 的数据以读为主，只支持单个写入者，并且写操作总是以添加的形式在文末追加，不支持在任意位置进行修改。

&emsp;

## HDFS 数据块 (Block)

每个磁盘都有默认的数据块大小，这是文件系统进行数据读写的最小单位。HDFS 同样也有数据块的概念，默认一个块（block）的大小为128MB（HDFS 的块这么大主要是为了最小化寻址开销），要在 HDFS 中存储的文件可以划分为多个分块，每个分块可以成为一个独立的存储单元。与本地磁盘不同的是，HDFS 中小于一个块大小的文件并不会占据整个 HDFS 数据块。对 HDFS 存储进行分块有很多好处：

- 一个文件的大小可以大于网络中任意一个磁盘的容量，文件的块可以利用集群中的任意一个磁盘进行存储。
- 使用抽象的块，而不是整个文件作为存储单元，可以简化存储管理，使得文件的元数据可以单独管理。
- 冗余备份。数据块非常适合用于数据备份，进而可以提供数据容错能力和提高可用性。每个块可以有多个备份（默认为三个），分别保存到相互独立的机器上去，这样就可以保证单点故障不会导致数据丢失。

&emsp;

## 集群架构

<img src="https://atts.w3cschool.cn/attachments/tuploads/hadoop/hdfs_architecture.jpg" title="" alt="HDFS架构" data-align="center">

HDFS 遵循主从架构，集群的节点分为两类：NameNode 和 DataNode，以**管理节点-工作节点**的模式运行，即一个 NameNode 和多个 DataNode，理解这两类节点对理解 HDFS 工作机制非常重要。

&emsp;

### NameNode

NameNode 作为管理节点，它负责整个文件系统的命名空间，并且维护着文件系统树和整棵树内所有的文件和目录，这些信息以两个文件的形式（命名空间镜像文件和编辑日志文件）永久存储在 NameNode 的本地磁盘上。

<img src="https://static001.infoq.cn/resource/image/66/e4/66817d43738191ecf24e05d4e37787e4.png" title="" alt="" data-align="center">

除此之外，NameNode 也记录每个文件中各个块所在的数据节点信息，全权管理数据块的复制，周期性地从集群中的每个 DataNode 接收心跳信号 (HeartBeat) 和块状态报告 (BlockReport)。接收到心跳信号意味着该 DataNode 节点工作正常。**块状态报告包含了一个该 DataNode 上所有数据块的列表**。NameNode 不永久存储块的位置信息，因为块的信息可以在系统启动时重新构建。此外，NameNode 还负责处理客户端读写请求。

<img src="https://img2018.cnblogs.com/blog/1608161/201907/1608161-20190709150608993-1616801523.png" title="" alt="" data-align="center">

由此可见，NameNode 作为管理节点，它的地位是非同寻常的，一旦 NameNode 宕机，那么所有文件都会丢失，因为 NameNode 是唯一存储了元数据、文件与数据块之间对应关系的节点，所有文件信息都保存在这里，NameNode 毁坏后无法重建文件。因此，必须高度重视 NameNode 的容错性。

为了使得 NameNode 更加可靠，Hadoop 提供了两种机制：

- 第一种机制是备份那些组成文件系统元数据持久状态的文件，比如：将文件系统的信息写入本地磁盘的同时，也写入一个远程挂载的网络文件系统（NFS），这些写操作实时同步并且保证原子性。
- 第二种机制是运行一个辅助 NameNode，用以保存命名空间镜像的副本，在 NameNode 发生故障时启用。



### DataNode

DataNode 作为文件系统的工作节点，根据需要存储并检索数据块，定期向 NameNode 发送他们所存储的块的列表。它需要根据客户端请求对文件系统执行读写操作，同时根据NameNode 的指令执行诸如块创建，删除和复制的操作。

&emsp;

### Secondary NameNode

它和 NameNode 并非主备关系，而是辅助 NameNode 进行合并 FsImage 和 EditLog 并起到备份作用。合并的过程会周期进行，一旦 NameNode 出现故障需要恢复，它可以更快的提供备份数据。

    &emsp;

## 数据复制

HDFS 被设计成能够在一个大集群中跨机器可靠地存储超大文件。它将每个文件存储成一系列的数据块，除了最后一个，所有的数据块都是同样大小的。为了容错，文件的所有数据块都会有副本。每个文件的数据块大小和副本系数都是可配置的。应用程序可以指定某个文件的副本数目。副本系数可以在文件创建的时候指定，也可以在之后改变。HDFS 中的文件都是一次性写入的，并且严格要求在任何时候只能有一个写入者。

副本的存放是 HDFS 可靠性和性能的关键，优化的副本存放策略是 HDFS 区分于其他大部分分布式文件系统的重要特性。在大多数情况下，副本系数是3，HDFS的存放策略是将一个副本存放在本地机架的节点上，一个副本放在同一机架的另一个节点上，最后一个副本放在不同机架的节点上。这种策略减少了机架间的数据传输，这就提高了写操作的效率。

&emsp;

## 读写过程

### 读过程

![](https://static001.infoq.cn/resource/image/7f/55/7f475f2076a42489c3ef6af11f827755.png)

1. 客户端调用 open 方法，打开一个文件。

2. 获取 block 的位置，即 block 所在的 DataNode，NameNode 会根据拓扑结构返回距离客户端最近的 DataNode。

3. 客户端直接访问 DataNode 读取 block 数据并计算校验和，整个数据流不经过 NameNode。

4. 读取完一个 block 会读取下一个 block。

5. 所有 block 读取完成，关闭文件。

&emsp;

### 写过程

![](https://static001.infoq.cn/resource/image/37/ca/37cdd8c78c5082227db26274c9a4f7ca.png)

1. 客户端调用 create 方法，创建一个新的文件；NameNode 会做各种校验，比如文件是否已经存在，客户端是否有权限等。

2. 如果校验通过，客户端开始写数据到 DataNode，文件会按照 block 大小进行切块，默认 128M（可配置），DataNode 构成 pipeline 管道，client 端向输出流对象中写数据，传输的时候是以比 block 更小的 packet 为单位进行传输，packet 又可拆分为多个 chunk，每个 chunk 都携带校验信息。

3. 每个 DataNode 写完一个块后，才会返回确认信息，并不是每个 packet 写成功就返回一次确认。

4. 写完数据，关闭文件。
