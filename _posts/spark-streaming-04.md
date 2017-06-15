---
title: Spark源码-streaming-04-几个重要的算子
date: 2017-04-12 20:59:48
tags:
    - spark
---
## 总结
1. DStream 的算子就是对 RDD 的巧妙组合
2. spark-streaming 部分结束

## updateStateByKey 算子
可以让我们为每个key维护一份state，并持续不断的更新该state，updateStateByKey操作，要求必须开启Checkpoint机制

1. 可以发现 DStream 里面是没有 updateStateByKey 方法的，这里用到了 DStream 伴生对象的一个隐式转换

          implicit def toPairDStreamFunctions[K, V](stream: DStream[(K, V)])
              (implicit kt: ClassTag[K], vt: ClassTag[V], ord: Ordering[K] = null):
            PairDStreamFunctions[K, V] = {
            new PairDStreamFunctions[K, V](stream)
          }

2. updateStateByKey 方法最后创建的是一个 StateDStream，StateDStream 的 compute  方法代码略多，就不贴代码了，下面简单介绍一下处理流程：

    - 尝试获取上一个 batch 创建的 StateDStream 创建的 RDD
    - 如果能够获取到则与当前的 RDD cogroup, 变为 ( key , CompactBuffer, CompactBuffer ), 然后执行状态更新函数
    - 如果不能获取到上一个 batch 创建的 rdd，则对当前 rdd 执行 groupByKey 算子，然后执行状态更新函数。
    - 注意这些 RDD 在得到后会判断 checkpointDuration, 然后设置 rdd checkpoint
    
          if (checkpointDuration != null && (time - zeroTime).isMultipleOf(checkpointDuration)) {
            newRDD.checkpoint()
            logInfo(s"Marking RDD ${newRDD.id} for time $time for checkpointing")
          }

## window 算子
对一个滑动窗口内的数据执行计算操作，包括 window，countByWindow，reduceByWindow，reduceByKeyAndWindow，countByValueAndWindow

其具体实现也是通过获取前面一段时间生成的 RDD, 然后多个 rdd 做 union

      override def compute(validTime: Time): Option[RDD[T]] = {
        val currentWindow = new Interval(validTime - windowDuration + parent.slideDuration, validTime)
        val rddsInWindow = parent.slice(currentWindow)
        Some(ssc.sc.union(rddsInWindow))
      }


## transform 算子
用于实现 DStream 到 RDD 的转化，这样就可以使用一些 RDD 上有，而 DStream 上没有的算子。下面是 TransformedDStream#compute 方法, transformFunc 就是我们传入处理函数。

      override def compute(validTime: Time): Option[RDD[U]] = {
        val parentRDDs = parents.map { parent => parent.getOrCompute(validTime).getOrElse(
          // Guard out against parent DStream that return None instead of Some(rdd) to avoid NPE
          throw new SparkException(s"Couldn't generate RDD from parent at time $validTime"))
        }
        // transformFunc 函数就是算子中传入的函数
        val transformedRDD = transformFunc(parentRDDs, validTime)
        if (transformedRDD == null) {
          throw new SparkException("Transform function must not return null. " +
            "Return SparkContext.emptyRDD() instead to represent no element " +
            "as the result of transformation.")
        }
        Some(transformedRDD)
      }
