# Structured streaming basic
* You get all of structured APIs 
* it has extra features such as end-to-end, exactly-once processing as well as fault-tolerance through checkpointing and writ-ahead logs
* input sources: kafka, file source, socket source for testing
* sinks: kafka, any file format, foreach sink (arbitrary computation on microbatch), console sink (testing), memory sink (debugging)
* output modes: how data is output
    * append: only new records to output sink
    * update: update changed records in place
    * complete: rewrite the full output path
* triggers: when data is output
    * by default, SS automatically look for new input records as soon as finishing processingthe last group of input data
    * this could generate many small files
    * spark supports triggers based on processing time
* Event-time processing: process data based on timestamps included in the record that may arrive out of order
    * watermarks: to specify how late systems can expect to see data in event time. (how long SS needs to remember old data)

# SS in actions
* `spark.sql.streaming.schemaInference` is an explicit option to enable schema inference
* action doesn't start until you start stream 

```scala
//define stream input
val streaming = spark.readStream.schema(dataSchema)
  .option("maxFilesPerTrigger", 1).json("data/activity-data")

//define transformation
val activityCounts = streaming.groupBy("gt").count()

//define output. in-memory sink. complete output mode, which means every trigger entire output table gets updated
val activityQuery = activityCounts.writeStream.queryName("activity_counts")
  .format("memory").outputMode("complete")
  .start()

// very important: keep the application up until you receive termination signal
activityQuery.awaitTermination()
```

# Transformations on Streams
* selections and filtering: all select and filter (where) transformation is supported
* aggregation: 
    * as of Spark2.2: multiple chained aggregations not supported (aggregation on streaming aggregation). work around is to write it out to intermediate sink
* joins: SS can join static DF with streaming DF
    * Spark 2.3, you can join multiple streams
    * as of spark2.2, full outer join, left join with stream on the right side, right joins with stream on the left side, not supported

# Input and Output (how, when, where)
## Where:
* file source and sink
    * `maxFilesPerTrigger`: you can control number of files that we read in during each trigger
    * any files added in to input directory for a streaming job need to appear atomically. SS will process partrially written files. In HDFS, you might have to move completed file to input directory
* kafka source and sink
    * Read:
        * `assign`, `subsribe`, `subscribePattern`: multiple ways to connect to kafka
        * multiple options: `staringOffsets`, `endingOffsets`, `failOnDataLoss`, `maxOffsetsPerTrigger`
        * each record will have schema that includes: key, value, topic, partition, offset, timestamp
    * Write:
        * have a topic column or specify destination topic as an option (`topic`)
* Foreach sink: similar to foreachPartitions in Dataset API. It allows arbitrary operation to be computed on each partition (in parallel)
    * implement `ForeachWriter` interface (open, process, close)
        * writer must be serializable
        * open method needs to do all initialization so that sink connection happens in executor
    * there are mechanism to gurantee exactly-once processing
        * `version`: monotonically increasing ID (increased per trigger)
        * `partitionId`: id of partrition of the output
        * open method can check those two (persist these info externally) and decide whether to process
* sources and sinks for testing: doesn't provide fault tolerance
    * socket (TCP) source: don't use it on production. socket is running on driver
    * console sink
    * memory sink: collect data to driver. for querying capabilities, it's recommended to have parquet file sink on distributed file system
## How data is output:
* append: append record to result table. default behavior. exactly-once gurantee
* complete: output entire state of result table to output sink. (useful for stateful data)
* update: only the rows that are different from previous write are written to sink
    * sink must support row-level updates
    * if query doesn't contain aggregation, this is equivalent to append mode
## When data is output:
* processing time trigger: specify duration as a string 

```scala
activityCounts.writeStream.trigger(Trigger.ProcessingTime("100 seconds"))
  .format("console").outputMode("complete").start()
```

* once trigger: useful for both dev and production. SS tracks input files proessed and computation state, it's easy to track your job status

```scala
import org.apache.spark.sql.streaming.Trigger

activityCounts.writeStream.trigger(Trigger.Once())
  .format("console").outputMode("complete").start()
```
