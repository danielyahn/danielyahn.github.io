# When to use RDD or Distributed Shared Variables (broadcast or accumulator)
* if you need:
    * very tight control over physical data placement across the cluster
    * to maintain some legacy codebase written using RDD
    * custom shared variable manipulation

# RDD
* get the sparkContext object from SparkSession: `spark.sparkContext`
* immutable, distributed dataset, records are just Java/Scala/Python object
    * reinvent the wheel for maniuplations
    * manual optimization for space and performance
    * manual optimization in Spark SQL such as reording filter and aggregation 
* it's trivial to switch between RDD and Datasets
* Python, could give you substantial amount of performance when using RDD: because you are running UDF row by row, and there's a lot of serialization cost between java and python processes

# Types of RDDs
* Many types, but mostly internal representations for DataFame API
* Two types: generic RDD vs key-value RDD
    * key-value RDD enables you to do custom partioning by key
* Formal definition of RDD
    1. list of partitions
    2. function for computing each split
    3. List of dependencies on other RDD
    4. (Optionally) Partitioner for key-value RDD
    5. (Optionally) list of preferred locations on which to compute each split (eg HDFS block location)

# Creating RDDs

```scala
spark.range(10).toDF() //dataset to dataframe
 .rdd //dataframe to rdd
 .map(rowObject => rowObject.getLong(0)) //get field from row

spark.range(10).rdd //dataset to rdd
  .toDF() //rdd to dataframe

// array of string to RDD
val myCollection = "Spark The Definitive Guide : Big Data Processing Made Simple"
  .split(" ")
val words = spark.sparkContext.parallelize(myCollection, 2)

//set name of rdd
words.setName("myWords")

//read file line by line
spark.sparkContext.textFile("/some/path/withTextFiles")
//read entire file into a record
spark.sparkContext.wholeTextFiles("/some/path/withTextFiles")
```

# Manipulating RDDs
* Similar to DataFrames, but you mainpulate raw Java or Scala objects instead of Spark types
* Transformations

```scala
words.distinct().collect() //returns unique array (RDD)

// filter
def startsWithS(individual:String) = {
  individual.startsWith("S")
}
words.filter(word => startsWithS(word)).collect()

// using column data as filter
val words2 = words.map(word => (word, word(0), word.startsWith("S")))
words2.filter(record => record._3).take(5)

words.flatMap(word => word.toSeq).take(5)
//sort words from longest to shortest (sort function sort in asc by default)
words.sortBy(word => word.length() * -1).take(2)

val fiftyFiftySplit = words.randomSplit(Array[Double](0.5, 0.5))
```

# Actions
* Reduce opeartion on partitions is not deterministic

```scala
spark.sparkContext.parallelize(1 to 20).reduce(_ + _) // 210

// if exceeds timeout, it will return imcomplete result
val confidence = 0.95
val timeoutMilliseconds = 400
words.countApprox(timeoutMilliseconds, confidence)

// hyperLogLog countApproxDistinct
words.countApproxDistinct(0.05)
words.countApproxDistinct(4, 10)

// it loads result set into driver! don't use it
words.countByValue()

// returns first value from rdd
words.first()
words.min()
words.max()

// take
words.take(5)
words.takeOrdered(5) //smallest 5
words.top(5) //bigest 5
val withReplacement = true
val numberToTake = 6
val randomSeed = 100L
words.takeSample(withReplacement, numberToTake, randomSeed)
```

# Saving files
* you are saving to plain-texts. 

```scala
words.saveAsTextFile("file:/tmp/bookTitle")
//compression
import org.apache.hadoop.io.compress.BZip2Codec
words.saveAsTextFile("file:/tmp/bookTitleCompressed", classOf[BZip2Codec])

// hadoop sequence file binary key-value pairs used in MapReduce
words.saveAsObjectFile("/tmp/my/sequenceFilePath")
```

# Cache and Persist
* cache: Persist RDD with the default storage level (MEMORY_ONLY).
* persist: Persist RDD with storageLevel (memory, disk, separate, off heap...)

# Checkpointing
* act of saving RDD to disk so that future references to this RDD point to those intermediate partitions on disk rather than recomputing the RDD from its original source.

# Per partition operations
* pipe command: execute external process once per partition, using RDD as piped input
* mapPartitions: per partition mapping function. `mapPartitionsWithIndex` provides index and iterator within each partition (good for debugging)
* foreachPartition: function has no return value. good way to write to source
* glom: converts each partition to array

```scala
//execute line count per partition
words.pipe("wc -l").collect()

// for each partition output 1
words.mapPartitions(part => Iterator[Int](1)).sum() // 2

def indexedFunc(partitionIndex:Int, withinPartIterator: Iterator[String]) = {
  withinPartIterator.toList.map(
    value => s"Partition: $partitionIndex => $value").iterator
}
words.mapPartitionsWithIndex(indexedFunc).collect()

//foreachPartition
words.foreachPartition { iter =>
  import java.io._
  import scala.util.Random
  val randomFileName = new Random().nextInt()
  val pw = new PrintWriter(new File(s"/tmp/random-file-${randomFileName}.txt"))
  while (iter.hasNext) {
      pw.write(iter.next())
  }
  pw.close()
}

//glom example
spark.sparkContext.parallelize(Seq("Hello", "World"), 2).glom().collect()
// Array(Array(Hello), Array(World))
```