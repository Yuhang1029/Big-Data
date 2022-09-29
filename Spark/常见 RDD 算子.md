# 常见 RDD 算子 (Operator)

**参考文档**

[RDD Programming Guide - Spark 3.3.0 Documentation](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-operations)

---

## Transformation 算子

### map

将 RDD 的数据一条条处理，处理的逻辑基于 map 算子中接收的处理函数，返回新的 RDD。

```python
# 定义一个方法，作为算子的传入函数体
def multiply(data):
    return data * 10

rdd.map(multiply)

# 定义 lambda 表达式来写匿名函数
rdd.map(lambda data : data * 10)
```

### flatMap

对 RDD 执行 map 操作，然后进行解除嵌套操作，即多维数组转成一维。

```python
rdd = sc.parallelize(["hadoop spark spark", "flink flink"])
# 假设我需要按照空格拆分得到每一个单词
rdd2 = rdd.map(lambda line : line.split(" "))
print(rdd2.collect())
# 得到的结果是 [["hadoop","spark", "spark"], ["flink", "flink"]]

# 如果希望最后得到是一个一维数组，应该用 flatMap
rdd2 = rdd.flatMap(lambda line : line.split(" "))
print(rdd2.collect())
# 得到的结果是 ["hadoop","spark", "spark", "flink", "flink"]
```

### reduceByKey

针对 K-V 型 RDD，自动按照 Key 分组，然后根据你提供的聚合逻辑，完成组内数据 (value) 的聚合操作。

```python
rdd = sc.parallelize([('a', 1), ('b', 1), ('a', 2), ('a', 1)])
print(rdd.reduceByKey(lambda a, b : a + b).collect())
```

### mapValues

针对二元元组的 RDD，对其内部的二元元组 Value 进行 map 操作。它和 map 基本一致，只是写法更加简便。

```python
rdd = sc.parallelize([('a', 1), ('b', 1), ('a', 2), ('a', 1)])
# 使用 map
rdd.map(lambda x : (x[0], x[1] * 10))
# 使用 mapValues
rdd.mapValues(lambda x : x * 10)
```

### groupBy

将 RDD 的数据进行分组。

```python
rdd = sc.parallelize([('a', 1), ('b', 1), ('a', 2), ('a', 1)])
# 通过 groupBy 进行分组
# groupBy 传入函数的意思是通过这个确定按照谁来分组
result = rdd.groupBy(lambda t : t[0])
print(result.collect())
```

### groupByKey

针对 K-V 型数据，自动按照 Key 来分组。

```python
rdd = sc.parallelize([('a', 1), ('b', 1), ('a', 2), ('a', 1)])
rdd2 = rdd.groupByKey()
# [('a', [1, 2, 1]), ('b', [1])]
print(rdd2.map(lambda x : (x[0], list(x[i]))).collect())
```

### filter

过滤想要的数据进行保留，传入一个单入参的函数，返回值是布尔型。

```python
rdd = sc.parallelize([0, 1, 2, 3, 4, 5])
# 通过 filter 过滤奇数
result = rdd.filter(lambda x : x % 2 == 1)
print(result.collect())
```

### distinct

对 RDD 数据去重，返回新的 RDD。

```python
rdd = sc.parallelize([0, 1, 1, 1, 2, 2, 3])
print(rdd.distinct().collect())
```

### union

把两个 RDD 合并成一个 RDD，RDD 类型不同也是可以合并的。

```python
rdd1 = sc.parallelize([0, 1, 1, 3])
rdd2 = sc.parallelize(['a', 'b', 'c'])
rdd3 = rdd1.union(rdd2)
# 输出 [0, 1, 1, 3, 'a', 'b', 'c']
print(rdd3.collect())
```

### join

对两个 RDD 执行 join 操作（可实现 SQL 的内/外连接），join 算子只能用于二元元组。

```python
rdd1 = sc.parallelize([('Jim', 1001), ('Tom', 1002), ('Bob', 1003)])
rdd2 = sc.parallelize([(1001, 'Department_1'), (1002, 'Department_2')])
# 通过 join 来进行数据关联
# 对于 join 算子来说，关联条件按照二元元组的key来进行关联
# [(1001, ('Jim', 'Department_1')), (1002, ('Tom', 'Department_2'))]
print(rdd1.join(rdd2).collect())

# 左外连接
# [(1001, ('Jim', 'Department_1')), (1002, ('Tom', 'Department_2')), (1003, ('Tom', None))]
print(rdd1.leftOuterJoin(rdd2).collect())
```

### intersection

求两个 RDD 的交集，返回一个新的 RDD。

```python
rdd1 = sc.parallelize([('a', 1), ('a', 3)])
rdd2 = sc.parallelize([('a', 1), ('b', 3)])
rdd3 = rdd1.intersection(rdd2)
# 输出 [('a', 1)]
print(rdd3.collect())
```

### glom

将 RDD 的数据加上嵌套，这个嵌套按照分区进行。

```python
# 两个分区 [[0, 1], [1, 3]]
rdd1 = sc.parallelize([0, 1, 1, 3], 2)
print(rdd1.glom().collect())

# 三个分区 [[0, 1], [1, 3], [4, 6]]
rdd2 = sc.parallelize([0, 1, 1, 3, 4, 6], 3)
print(rdd2.glom().collect())

# 如果希望解嵌套可以利用 flatMap 同时传入一个空的实现方法
# 打印结果就是 [0, 1, 1, 3, 4, 6]
print(rdd2.glom().flatMap(lambda x: x).collect())
```

### sortBy

对 RDD 数据进行排序，基于你指定的排序依据。

```python
rdd = sc.parallelize([('a', 1), ('b', 1), ('a', 2), ('a', 3), ('b', 3)])
# 按照 Value 数字进行排序，参数二表示升序降序，默认升序
# [('a', 1), ('b', 1), ('a', 2), ('a', 3), ('b', 3)]
print(rdd.sortBy(lambda x : x[1], ascending=True, numPartition=1).collect())

# 按照 Key 字母进行排序
# [('', 1), ('b', 1), ('a', 2), ('a', 3), ('b', 3)]
print(rdd.sortBy(lambda x : x[0], ascending=False, numPartition=1).collect())    
```

### sortByKey

针对 K-V 型数据，按照 Key 进行排序。sortByKey 可以传入一个函数在排序前对 Key 进行预处理。

```python
rdd = sc.parallelize([('A', 1), ('b', 1), ('a', 2), ('a', 3), ('b', 3)])
# 忽略大小写排序
# [('A', 1), ('a', 2), ('a', 3), ('b', 1), ('b', 3)]
rdd2 = rdd.sortByKey(ascending=True, numPartitions=1, keyfunc=lambda k : str(k).lower())
print(rdd2.collect())
```

&emsp;

## Action 算子

### countByKey

统计 Key 出现的次数，适用于 K-V 型 RDD。

```python
rdd = sc.textFile("../words.txt")
rdd2 = rdd.flatMap(lambda x : x.split(' ')).map(lambda x : (x, 1))
result = rdd2.countByKey()
# 输出一个 Python 的类 Dict
print(result)
```

### collect

将 RDD 各个分区内的数据，统一收集到 Driver 中，形成一个 List 对象。注意使用前需要自己确认结果数据集不会很大。

### reduce

对 RDD 数据按照你传入的逻辑进行聚合，得到最终的结果。

```python
rdd = sc.parallelize([0, 1, 2, 3, 4])
print(rdd.reduce(lambda a, b : a + b))
```

### first / take

取出 RDD 的第一个元素 / 取出前n个元素组成 List。

```python
# 1
print(sc.parallelize([3, 2, 1]).first())
# [3, 2, 1]
print(sc.parallelize([3, 2, 1, 0, 3]).take(3))
```

### top

对 RDD 数据进行降序排序，取出前n个。

```python
# [8, 5]
print(sc.parallelize([0, 5, 8, 3, 2, 1]).top(2))
```

### count

返回 RDD 中有几条数据。

### takeSample

随机抽样 RDD 数据

```python
rdd = sc.parallelize([3, 2, 1, 5, 7, 2, 1, 1, 5])
# 第一个变量代表是否允许抽取相同位置的数据
print(rdd.takeSample.(False, 4))
```

### takeOrdered

对 RDD 排序并且取出前n个，可以正向排序或逆向排序。

```python

```
