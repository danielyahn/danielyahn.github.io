# Key-value basics

```scala
//simple way to create key
// (first letter of the word, word)
val keyword = words.keyBy(word => word.toLowerCase.toSeq(0).toString)

// map on values only
keyword.mapValues(word => word.toUpperCase).collect()
// expand rows for each string (each character as separate record)
keyword.flatMapValues(word => word.toUpperCase).collect()

// extracring keys.values
keyword.keys.collect()
keyword.values.collect()

// look up values by key
keyword.lookup("s")
```

# Aggregations
* countByKey: count number of elements for each key, return to driver
* groupByKey: each executor must hold all values for a given key in memory before applying function to them. Be careful with memory management.
* reduceByKey: reduce happens in each partition (no shuffle)

```scala
val timeout = 1000L //milliseconds
val confidence = 0.95
KVcharacters.countByKey()
KVcharacters.countByKeyApprox(timeout, confidence)

KVcharacters.groupByKey().map(row => (row._1, row._2.reduce(addFunc))).collect()

KVcharacters.reduceByKey(addFunc).collect()
```

# Other (Unused) aggregation methods
* aggregate: first function aggregates within partition, second function aggregates across partition. final aggregation on the driver (potential memory issue).
* treeAggregate: pushes down subaggregation to executor (so that driver doesn't run out of memory)
* aggregateByKey: not by partition, but by key
* combineByKey: not aggregate, but combines values 
* foldByKey: merges values for each key using associative function

```scala
nums.aggregate(0)(maxFunc, addFunc)

val depth = 3
nums.treeAggregate(0)(maxFunc, addFunc, depth)

val valToCombiner = (value:Int) => List(value)
val mergeValuesFunc = (vals:List[Int], valToAppend:Int) => valToAppend :: vals
val mergeCombinerFunc = (vals1:List[Int], vals2:List[Int]) => vals1 ::: vals2
// now we define these as function variables
val outputPartitions = 6
KVcharacters
  .combineByKey(
    valToCombiner,
    mergeValuesFunc,
    mergeCombinerFunc,
    outputPartitions)
  .collect()

KVcharacters.foldByKey(0)(addFunc).collect()
```

# Joina
* CoGroups: group together up to three key-value RDDs together
* inner join: you have option to specify number of output partitions
* zip: zip together two RDDs to pairRDD. You need to have same # of partitions and elements.

```scala
//cogroup example (key, (v1,v2,v3))
import scala.util.Random
val distinctChars = words.flatMap(word => word.toLowerCase.toSeq).distinct
val charRDD = distinctChars.map(c => (c, new Random().nextDouble()))
val charRDD2 = distinctChars.map(c => (c, new Random().nextDouble()))
val charRDD3 = distinctChars.map(c => (c, new Random().nextDouble()))
charRDD.cogroup(charRDD2, charRDD3).take(5)

//inner join
val keyedChars = distinctChars.map(c => (c, new Random().nextDouble()))
val outputPartitions = 10
KVcharacters.join(keyedChars).count()
KVcharacters.join(keyedChars, outputPartitions).count()

//zip
val numRange = sc.parallelize(0 to 9, 2)
words.zip(numRange).collect()
```

# Controlling Partitions
* coalesce: collapes partitions on the same worker in order to avoid a shuffle when repartioning
* repartition: repartition up and down, but performs shuffle
* repartitionAndSortWithinPartitions: option to repartition, and sort each partition
## Custom Partitioning
* primary reason for using RDD, avoid data skew!
* you need to implement your own class that extends `Partitioner`
* 2 Partitioner:
    * discrete: `HashPartitioner`
    * continous: `RangePartitioner`

```scala
// in Scala
import org.apache.spark.Partitioner
class DomainPartitioner extends Partitioner {
 def numPartitions = 3
 def getPartition(key: Any): Int = {
   val customerId = key.asInstanceOf[Double].toInt
   if (customerId == 17850.0 || customerId == 12583.0) {
     return 0
   } else {
     return new java.util.Random().nextInt(2) + 1
   }
 }
```

# Custom Serialization
* use Kyro serialization
* `spark.serializer=org.apache.spark.serializer.KyroSerializer`
* kyro's used by default since 2.0, when shuffuling RDD with simple types, arrays, string
* you have to register custom classes