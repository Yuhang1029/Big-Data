# MapReduce 简介

**参考文档**

[MapReduce核心思想【图文介绍】](https://www.itheima.com/news/20211203/180449.html)

---

## 设计思想

MapReduce 的设计是为了应对大数据的场景，对于相互间不具有依赖关系的计算任务，实现并行计算的策略。具体来说就是首先通过 Map 阶段拆分数据，把大数据拆分成若干个小的数据，多个程序同时并行计算产生中间结果，然后通过 Reduce 来进行聚合，通过计算汇总得到最终结果。

MapReduce 作为一种**分布式计算模型**，它主要用于解决海量数据的计算问题。用户不需要考虑底层的计算细节，只需要专注使用场景的逻辑。使用MapReduce 分析海量数据时，每个 MapReduce 程序被初始化为一个工作任务，**每个工作任务可以分为Map和Reduce两个阶段**，如下图：

<img src="http://www.itheima.com/images/newslistPIC/1638525886817_MapReduce%E6%A0%B8%E5%BF%83%E6%80%9D%E6%83%B3.jpg" title="" alt="1638525886818_MapReduce核心思想.jpg" data-align="center">MapReduce 的优点包括：

* MapReduce 框架提供了二次开发的借口。简单地实现了一些接口，就可以完成一个分布式程序。

* 良好的扩展性。当计算机资源得不到满足的时候，可以通过增加机器来扩展它的计算能力，这个是处理海量数据的关键。

* 高容错性，任何一个机器宕机可以把其计算任务转移到其他节点。

MapReduce 的局限性：

* 实时计算性能差，主要用于离线任务。

* 主要时候静态数据，不适合流式数据。

&emsp;

## 实例进程

一个完整的 MapReduce 程序在分布式运行时有三类：

* MRAppMaster：负责整个 MapReduce 过程调度和状态协调。

* MapTask：负责 Map 阶段的整个数据处理流程。

* ReduceTask：负责 Reduce 阶段的整个数据处理流程。

运行时候 Map 和 Reduce 都是同时出现，如果需要多个步骤就会串行执行。

&emsp;

## 执行细节

### Map 过程

1. 首先会把输入目录下的文件按照一定标准进行逻辑切片，默认切片大小和数据的块大小一致（128M），每一个切片交给 MapTask 处理。

2. 随后对切片中的数据按照一定规则读取，返回 <K, V> 键值对。这个是通过 `InputFormat` 来控制，如果不额外自定义的话 Hadoop 会使用默认的 `TextInput`，会按行读取，`key` 是每一行起始位置的偏移量，一般不会使用，`value` 是本行的文本内容。

3. 最后会调用 Mapper 类中的 `map()` 方法，每一个键值对会调用一次。

4. Map 的输出数据同样是键值对，会写入内存缓冲区，达到比例就溢出到磁盘上。

5. 对所有溢出文件进行最终合并，成为一个文件。

&emsp;

### Reduce 过程

1. ReduceTask 会主动从 MapTask 复制拉取属于自己需要处理的数据。

2. 把拉取来的数据进行合并，随后排序。

3. 键值相等的键值对会调用一次 Reduce 方法，最后把输出写到 HDFS 中。

&emsp;

## 代码示例

Mapper 示例

```java
public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
        private final static IntWritable one = new IntWritable(1);
        private final Text word = new Text();

        public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            StringTokenizer itr = new StringTokenizer(value.toString());
            while (itr.hasMoreTokens()) {
                word.set(itr.nextToken());
                context.write(word, one);
            }
        }
    }
```

`TokenizerMapper` 是自定义的 Mapper 类，它需要继承原有的 `Mapper`，同时在泛型里指定输入输出类型。输入在前面说过，一般 Key 是 Character Offset，Value 就是对应的每一行信息，这里的 `Text` 是 `Hadoop` 自己的实现类，可以转化成 `String` 去处理。需要注意的是，无论是输入还是输出都需要用 `Text`，中间的实际逻辑可以用 `String` 处理。`map()` 函数这里处理的就是这个对应的逻辑，最后需要写入 `context`。

&emsp;

Reducer 示例

```java
public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
        private IntWritable result = new IntWritable();

        public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
            int sum = 0;
            for (IntWritable val : values) {
                sum += val.get();
            }
            result.set(sum);
            context.write(key, result);
        }
    }
```

`IntSumReducer` 是自定义的 Reducer 类，它需要继承原有的 `Reducer`，输入的 `K-V` 类型和 `Mapper` 的输出类型是一致的。reduce() 函数是整个核心逻辑函数，如果说 `Mapper` 的输出是 `K-V`，这里拿到的输入就是 `K-list<V>`，随后可以进行相应的逻辑处理。

&emsp;

Main() 示例

```java
public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class)
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
}
```

通过反射设置 `Mapper` 和 `Reducer`，确定最后的输出 `K-V` 类型。
