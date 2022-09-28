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
rdd = sc.parallelize(["hadoop spark spark", "flink flink"])
```


