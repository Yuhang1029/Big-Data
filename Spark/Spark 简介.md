# Spark 简介

参考文档

[大数据基础：Spark工作原理及基础概念](https://www.infoq.cn/article/zk8eyph0wn5xuywazstj)

[剖析Spark数据分区之Spark RDD分区 - 掘金](https://juejin.cn/post/6844904021502001166)

---

## 什么是 Spark

Spark 是 UC Berkeley AMP Lab 开源的通用分布式并行计算框架，目前已成为 Apache 软件基金会的顶级开源项目。现如今， Spark 已经是大数据领域的必备计算引擎，基本替代了传统的 MapReduce 离线计算，在机器学习和 AI 方面得到了显著发展。

相比较传统的 Hadoop，spark 有着以下几个优势：

* 高性能。Spark 具有 Hadoop MR 所有的优点，Hadoop MapReduce 每次计算的中间结果都会存储到 HDFS 的磁盘上，而 Spark 的中间结果可以保存在内存，在内存中进行数据处理。

* 高容错。基于血统 (Lineage) 的数据恢复：Spark 引入了弹性分布式数据集 RDD 的抽象，它是分布在一组节点中的只读的数据的集合，这些集合是弹性的且是相互依赖的，如果数据集中的一部分的数据发生丢失可以根据血统关系进行重建。

* 多种语言支持，包括 Java、Python、R 和 Scala。

* 支持多种的存储介质，在存储层 Spark 支持从 HDFS，Hive，AWS 等读入和写出数据，也支持从 HBase，ES 等大数据库中读入和写出数据，同时也支持从 mysql 等关系型数据库中读入写出数据，在实时流计算在可以从 flume，Kafka 等多种数据源获取数据并执行流式计算。

* Hadoop 中 MapReduce 中间计算结果存在 HDFS 上，延迟大；而 Spark 利用 RDD 在内存中，延迟小。

尽管 Spark 相比 Hadoop 有着较大的优势，但是并不能完全取代 Hadoop：

* 在计算层面，尽管有比较大的性能优势，但是至今仍有许多计算工具基于 MR 架构，比如非常成熟的 Hive。

* Spark 仅仅是计算，而 Hadoop 生态圈除了计算还有存储 (HDFS) 和资源管理调度 (Yarn)。

&emsp;

## Spark 名词合集

<img src="https://spark.apache.org/docs/latest/img/cluster-overview.png" title="" alt="Spark cluster components" data-align="center">

* Application：应用，就是程序员编写的 Spark 程序。

* Driver：驱动，用来执行 `main` 函数的 JVM 进程和新建 `SparkContext`。

* SparkContext：Spark 运行时的上下文环境，用来和 `ClusterManager` 进行通信，并进行资源的请求，任务的分配和监控等。

* ClusterManager：集群管理器。对于 Standalone 模式就是 Master，对于 Yarn 模式就是 ResourceManager/ApplicationMaster，在集群上做统一的资源管理。

* Worker：工作节点。是拥有 CPU / 内存等资源的机器，是真正干活的节点。

* Executor：运行在 Worker 中的 JVM 进程。

* Task：一个分区上的一系列操作就是一个 Task，并行执行，可以理解成 Task (线程) 是运行在 Executor (进程) 中的最小单位。

* TaskSet：任务集，就是同一个 Stage 中各个 Task 组成的集合。

&emsp; 

## Spark 核心概念

### RDD (Resilient Distributed Datasets)

RDD 是弹性分布式数据集，是一种不可变的，容错的、可以被并行操作的元素集合，是 Spark 对所有数据处理的一种基本抽象。它是一个分区的集合，一个 RDD 有一个或者多个分区，分区的数量决定了 Spark 任务的并行度。

在 RDD 之前，如果想做类型 WordCount 的大数据计算，有两种选择，一个是使用 Java/Scala 中的原生 List，但是因为 List 只支持单机版，如果想要做分布式计算，需要做很多额外的事情例如进程通信，负载均衡，非常麻烦，由此诞生了框架；另一种是使用 Hadoop 中的 MapReduce，但是运行效率很低，很早就淘汰了。

由于 RDD 的数据量很大，因此为了计算方便，需要将 RDD 进行切分并存储在各个节点的分区当中，从而当我们对 RDD 进行各种计算操作时，实际上是对每个分区中的数据进行并行的操作。也就是一份待处理的原始数据会被按照相应的逻辑切分成多分。

RDD 主要有五个特征：

* RDD 是有分区的，**一份 RDD 的数据本质上是分割成了多个分区，分区是 RDD 数据存储的最小单位**。RDD 是逻辑对象，分区实际上是物理对象。

* RDD 的方法会作用在其所有的分区之上，RDD 在任务计算时是以分区为单位的。

* RDD 之间存在血缘关系 (lineage)。每个 RDD 都有依赖关系（源RDD的依赖关系为空）。

* K-V 型的 RDD 可以有分区器，默认 Hash 分区。这个特性是可选的。

* RDD 的分区规划，会尽量靠近数据所在的服务器上，因为这样可以本地读取，避免网络读取。

&emsp;

<img src="https://static001.infoq.cn/resource/image/29/b0/29f7da8da6498a46795863522b42beb0.png" title="" alt="" data-align="center">

RDD 间存在着血统继承关系，其本质上是 RDD 之间的依赖关系。依赖有两种类型：

- 宽依赖 (Shuffle Dependency)：存在 Shuffle 操作的依赖关系，即父 RDD 的一个分区会被多个子 RDD 的分区依赖。

- 窄依赖 (Narrow Dependency)：即父 RDD 与子 RDD 之间的分区 (partition) 是一一对应的，换句话说，父 RDD 中，一个分区内的数据是不能被分割的，只能由子 RDD 中的一个分区整个利用。

&emsp;

Spark 对于窄依赖进行优化，可以通过合并形成管道 (pipeline)，同一个管道中的各个操作可以由一个线程执行完，如果有一个分区数据丢失，只需要从父 RDD 的对应分区重新计算即可，不需要重新计算整个任务，提高容错。

&emsp;

#### RDD 的创建

RDD 的创建方式主要有两种：

* 通过并行化集合创建，把本地对象转成分布式 RDD。

```python
rdd = sc.parallelize([1,2,3,4,5],4)
print(rdd.getNumPartitions())
print(rdd.collect())
```

* 读取文件创建，通过 textFile API 既可以读取本地数据，也可以读取 HDFS 数据。

```python
rdd = sc.textFile("~/Desktop/1.txt", 5)
```

&emsp;

#### RDD 算子

分布式集合对象上的 API 称为算子，可以通过一系列的算子对 RDD 进行操作，主要分 Transformation 和 Action 两种。

- Transformation (转换)：是对已有的 RDD 进行换行生成新的 RDD，对于转换过程采用惰性计算机制，如果没有 Action 算子，不会立即计算出结果。常用的方法有 map，filter，flatMap 等。

- Action (执行)：对已有对 RDD 对数据执行计算产生结果，并将结果返回 Driver 或者写入到外部存储中，返回值不再是 RDD。常用到方法有 reduce，collect，saveAsTextFile 等。

对于这两类算子来说，Transformation 算子相当于构建执行计划，Action 算子相当于让这个计划执行。

&emsp;

### DAG (Directed Acyclic Graph)

DAG 是有向无环图，在 Spark 中， 使用 DAG 来描述我们的计算逻辑，是 Spark 任务执行的流程图。DAG 的开始就是从创建 RDD 开始，到 Action 结束。一个 Spark 程序有几个 Action 操作就有几个 DAG。

Spark 包括 DAG Scheduler 和 Task Scheduler。

![](https://static001.infoq.cn/resource/image/e3/b3/e3d8a6c80cbc6b71134e98e37b37a1b3.png)

DAG Scheduler 是面向 Stage 的高层级的调度器，DAG Scheduler 把 DAG 拆分为多个 Task，每组 Task 都是一个 Stage，解析时是以 shuffle 为边界进行反向构建的，每当遇见一个 shuffle，spark 就会产生一个新的 stage，接着以 TaskSet 的形式提交给底层的调度器（Task Scheduler），每个 Stage 封装成一个 TaskSet。DAG Scheduler 需要记录 RDD 被存入磁盘物化等动作，同时会需要 Task 寻找最优等调度逻辑，以及监控因 shuffle 跨节点输出导致的失败。

![](https://static001.infoq.cn/resource/image/d5/17/d58074f34dc8b8d02f86a7ea023c4917.png)

Task Scheduler 负责每一个具体任务的执行。它的主要职责包括：

- 任务集的调度管理；

- 状态结果跟踪；

- 物理资源调度管理；

- 任务执行；

- 获取结果。

&emsp;

RDD 的容错机制就是通过将 RDD 间转移操作所构建的 DAG 来实现的，抽象来看，

### Stage

DAG Scheduler 会把 DAG 切割成多个相互依赖的 Stage，划分 Stage 的一个依据是 RDD 间的宽窄依赖。在对 Job 中的所有操作划分 Stage 时，一般会按照倒序进行，即从 Action 开始，遇到窄依赖操作，则划分到同一个执行阶段，遇到宽依赖操作，则划分一个新的执行阶段，且新的阶段为之前阶段的 parent，然后依次类推递归执行。子 Stage 需要等待所有的 父 Stage 执行完之后才可以执行，这时 Stage 之间根据依赖关系构成了一个大粒度的 DAG。在一个 Stage 内，所有的操作以串行的 pipeline 的方式，由一组 Task 完成计算，不同 Task 之间并行执行，无需等待。

### Job

作业，按照 DAG 执行就形成了 RDD 的执行流程图。

![](https://static001.infoq.cn/resource/image/e8/ee/e83d7957e11ecfacfda1e450b632c2ee.png)

Job 的整体调度流程包含以下几步：

1. Driver 启动并且注册 SparkContext。

2. SparkContext 向 ClusterManager 注册并申请资源。

3. ClusterManager 找到 Worker 并且分配相应资源，启动 Executor。

4. SparkContext 构建 DAG 图。

5. DAG Scheduler 将 DAG 图分解成多个 Stage，同一个 Stage 中的多个 Task 组成 TaskSet，多个 Task 并行执行。

6. Task Scheduler 提交并监控 Task。

7. 注销资源。
