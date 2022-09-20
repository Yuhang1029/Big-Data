# YARN 简介

**参考文档**

[Yarn快速入门系列(1)——基本架构与三大组件介绍-云社区-华为云](https://bbs.huaweicloud.com/blogs/302284)

---

## 什么是 YARN

在古老的 Hadoop1.0 中，MapReduce 的 JobTracker 负责了太多的工作，包括资源调度，管理众多的 TaskTracker 等工作。这自然是不合理的，于是 Hadoop 在 1.0 到 2.0 的升级过程中，便将 JobTracker 的资源调度工作独立了出来，而这一改动，直接让 Hadoop 成为大数据中最稳固的那一块基石，而这个独立出来的资源管理框架，就是 YARN。

Apache Hadoop YARN （Yet Another Resource Negotiator，另一种资源协调者）是一种新的 Hadoop 资源管理器，它是一个**通用资源管理系统和调度平台**，可为上层应用提供统一的资源管理和调度。它的引入为集群在利用率、资源统一管理和数据共享等方面带来了巨大好处。也就是说 YARN 在 Hadoop 集群中充当资源管理和任务调度的框架。MapReduce 等运算程序则相当于运行于操作系统之上的应用程序，YARN 为这些程序提供运算所需的资源 （cpu，内存）。

&emsp;

## 基本架构

<img src="https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/yarn_architecture.gif" title="" alt="MapReduce NextGen Architecture" data-align="center">

YARN是一个资源管理、任务调度的框架，主要包含三大模块：ResourceManager (RM), NodeManager (NM), ApplicationMaster (AM)。

其中：

        ResourceManager 负责所有资源的监控、分配和管理，一个集群只有一个；

        NodeManager 负责每一个节点的维护，一个集群有多个。

        ApplicationMaster 负责每一个具体应用程序的调度和协调，一个集群有多个；

对于所有的applications，**ResourceManager 拥有绝对的控制权和对资源的分配权，而每个 ApplicationMaster 则会和 ResourceManager 协商资源，同时和 NodeManager 通信来执行和监控task**。

&emsp;

### ResourceManager

ResourceManager 是 YARN中 的主节点服务，它负责集群中所有资源的统一管理和作业调度。NodeManager 以心跳的方式向 ResourceManager 汇报资源使用情况（目前主要是CPU 和内存的使用情况）。ResourceManager 只接受 NodeManager 的资源回报信息，对于具体的资源处理则交给NM自己处理。

&emsp;

### NodeManager

NodeManager 是每个节点上的资源和任务管理器，它是管理这台机器的代理，负责该节点程序的运行，以及该节点资源的管理和监控。YARN 集群每个节点都运行一个NodeManager。NodeManager 定时向 ResourceManager 汇报本节点资源（CPU、内存）的使用情况和 Container 的运行状态。当 ResourceManager 宕机时 NodeManager 自动连接 ResourceManager 备用节点。NodeManager 接收并处理来自 ApplicationMaster 的Container 启动、停止等各种请求。

注意在这里，Container 是 YARN中 的资源抽象，它封装了某个节点上的多个维度的资源，如 CPU、内存、磁盘、网络等。当 ApplicationMaster 向 ResourceManager 申请资源时，ResourceManager 为 ApplicationMaster 返回的资源是用 Container 表示的。

&emsp;

### ApplicationManager

每当 Client 提交一个 Application 时候，就会新建一个 ApplicationMaster 。由这个 ApplicationMaster 去与 ResourceManager 申请容器资源，获得资源后会将要运行的程序发送到容器上启动，然后进行分布式计算。

这里可能有些难以理解，为什么是把运行程序发送到容器上去运行？如果以传统的思路来看，是程序运行着不动，然后数据进进出出不停流转。但当数据量大的时候就没法这么玩了，因为海量数据移动成本太大，时间太长。但是中国有一句老话山不过来，我就过去。大数据分布式计算就是这种思想，既然大数据难以移动，那我就把容易移动的应用程序发布到各个节点进行计算呗，这就是大数据分布式计算的思路。

&emsp;

## 提交一个 Application 到 YARN

![](/Users/yuhangliu/Library/Application%20Support/marktext/images/2022-09-19-22-30-32-image.png)

这张图简单地标明了提交一个程序所经历的流程，接下来我们来具体说说每一步的过程。

1. Client 向 Yarn 提交 Application，这里我们假设是一个 MapReduce 作业。
2. ResourceManager 向 NodeManager 通信，为该 Application 分配第一个容器。并在这个容器中运行这个应用程序对应的 ApplicationMaster。
3. ApplicationMaster 启动以后，对作业（也就是 Application）进行拆分，拆分 task 出来，这些 task 可以运行在一个或多个容器中。然后向 ResourceManager 申请要运行程序的容器，并定时向 ResourceManager 发送心跳。
4. 申请到容器后，ApplicationMaster 会去和容器对应的 NodeManager 通信，而后将作业分发到对应的 NodeManager 中的容器去运行，这里会将拆分后的 MapReduce 进行分发，对应容器中运行的可能是 Map 任务，也可能是 Reduce 任务。
5. 容器中运行的任务会向 ApplicationMaster 发送心跳，汇报自身情况。当程序运行完成后， ApplicationMaster 再向 ResourceManager 注销并释放容器资源。


