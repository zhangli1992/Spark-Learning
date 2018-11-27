来源:https://cloud.tencent.com/developer/article/1043490
#### 1.背景:
combineByKey()是一种最常用对基于键进行聚合对函数。大多数基于键聚合的函数都是用它实现。跟aggregate()一样，combineByKey()可以让用户返回与输入数据类
型不同的返回值。
其定义如下:
```scala
def combineByKey[C](
    //在找到给定分区中第一次碰到的key（在RDD元素中）时被调用。此方法为这个key初始化一个累加器。
    createCombiner : V => C,
    //当累加器已经存在的时候（也就是上面那个key的累加器）调用。
    mergeValue : (C,V) => C,
    // 如果哪个key跨多个分区，该参数就会被调用。
    mergeCombiners : (C,C) => C,
    partitioner : Partitioner,
    mapSideCombine : Boolean = true,
    serializer : Serializer = null
    ): RDD[(K,C)] = { //实现略 }
 ```
函数式风格与命令式风格不同之处在于它说明了代码做了什么（what to do），而不是怎么做(how to do)。combineByKey函数主要接受了三个函数作为参数，分别为
createCombiner、mergeValue、mergeCombiners。这三个函数足以说明它究竟做了什么。理解了这三个函数，就可以很好地理解combineByKey。

#### 2.原理:
由于combineByKey()会遍历分区中的所有元素，因此每个元素的键要么还没有遇到过，要么就和之前的某个元素的键相同。
1. 如果这是一个新的元素，combineByKey()会使用一个叫作createCombiner()的函数来创建那个键对应的累加器的初始值。需要注意的是，这一过程会在每个
   分区中第一次出现各个键时发生，而不是在整个RDD中第一次出现一个键时发生。
2. 如果这是一个在处理当前分区之前已经遇到的键，它会使用mergeValue()方法将该键的累加器对应的当前值与这个新的值进行合并。
3. 由于每个分区都是独立处理的，因此对于同一个键可以有多个累加器。如果有两个或者更多的分区都有对应同一个键的累加器，就需要使用用户提供的
   mergeCombiners()方法将各个分区的结果进行合并。

#### 3.示例:
// 关闭 spark-shell INFO/DEBUG 调试信息
```scala
scala> sc.setLogLevel("WARN")

scala> val inputrdd = sc.parallelize(Seq(
                        ("maths", 50), ("maths", 60),
                        ("english", 65),
                        ("physics", 66), ("physics", 61), ("physics", 87)), 
                        1)
inputrdd: org.apache.spark.rdd.RDD[(String, Int)] = ParallelCollectionRDD[41] at parallelize at <console>:27

scala> inputrdd.getNumPartitions                      
res55: Int = 1

scala> val reduced = inputrdd.combineByKey(
         (mark) => {
           println(s"Create combiner -> ${mark}")
           (mark, 1)
         },
         (acc: (Int, Int), v) => {
           println(s"""Merge value : (${acc._1} + ${v}, ${acc._2} + 1)""")
           (acc._1 + v, acc._2 + 1)
         },
         (acc1: (Int, Int), acc2: (Int, Int)) => {
           println(s"""Merge Combiner : (${acc1._1} + ${acc2._1}, ${acc1._2} + ${acc2._2})""")
           (acc1._1 + acc2._1, acc1._2 + acc2._2)
         }
     )
reduced: org.apache.spark.rdd.RDD[(String, (Int, Int))] = ShuffledRDD[42] at combineByKey at <console>:29

scala> reduced.collect()
Create combiner -> 50
Merge value : (50 + 60, 1 + 1)
Create combiner -> 65
Create combiner -> 66
Merge value : (66 + 61, 1 + 1)
Merge value : (127 + 87, 2 + 1)
res56: Array[(String, (Int, Int))] = Array((maths,(110,2)), (physics,(214,3)), (english,(65,1)))

scala> val result = reduced.mapValues(x => x._1 / x._2.toFloat)
result: org.apache.spark.rdd.RDD[(String, Float)] = MapPartitionsRDD[43] at mapValues at <console>:31

scala> result.collect()
res57: Array[(String, Float)] = Array((maths,55.0), (physics,71.333336), (english,65.0))
```

注意：本例中因为只有一个分区所以 mergeCombiners 并没有用到，你也可以通过下面的代码从另外角度来验证：
```scala
scala> var rdd1 = sc.makeRDD(Array(("A",1),("A",2),("B",1),("B",2),("C",1)))
rdd1: org.apache.spark.rdd.RDD[(String, Int)] = ParallelCollectionRDD[64] at makeRDD at :21

scala> rdd1.getNumPartitions    
res18: Int = 64 

scala> rdd1.combineByKey(
            (v : Int) => v + "_",   
            (c : String, v : Int) => c + "@" + v,  
            (c1 : String, c2 : String) => c1 + "$" + c2
            ).collect
res60: Array[(String, String)] = Array((A,2_$1_), (B,1_$2_), (C,1_))
```
在此例中，因为分区多而记录少，可以看做每条记录都跨分区了，所以没有机会用到 mergeValue，最后直接 mergeCombiners  得到结果 。
除了可以进行group、average之外，根据传入的函数实现不同，我们还可以利用combineByKey完成诸如aggregate、fold等操作。这是一个高度的抽象，但从声明的
角度来看，却又不需要了解过多的实现细节。这正是函数式编程的魅力。
