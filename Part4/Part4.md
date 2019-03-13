# Part IV - Production Applications
* Execution modes: cluster, client, local
* Life Cycle: (Session - Job - Stage - Task)
    * 1 spark Session
    * 1 or many spark Jobs for spark Session: one Spark job for one action
    * 1 or more Stages for Jobs: groups of tasks that can be executed together to compute the same operation on multiple machines without shuffle
    * 1 or more Tasks for Stages: combination of blocks of data and a set of transformations that will run on a single executor
* pipelining: at/below RDD level, collapse steps into a single stage
* shuffle:
    * first step, source tasks write shuffle files to their local disks
    * if you run another Job on the smae data, it will skip pre-shuffle stages since data is already persisted
    * More efficient to use DataFrame or RDD cache method
* Job scheduling within an application
    * multiple parallel jobs can run simultaenously if submitted from separate threads
    * runs in FIFO by default
    * `spark.scheduler.mode: Fair` supports pools and setting priorities

## Spark UI
* Spark REST API can be used for custom monitoring
* Spark UI History Server:
    * reconstruct Spark UI and REST API after SparkContext finishes
    * `event log` needs to be configured (`spark.ecentLog.dir`)
    * history server is a standalone application

## Debugging and Spark First Aid
* Slowness with certain task "straggler"
    * try repartitioning by another combination of columns
    * increase memory allocated to executor
    * you might have unhealthy executor/node
    * try to convert UDFs into DataFrame code
    * ensure UDAF (user-defined aggergation functions) is running on a small enough batch of data
    * turn on `speculation`: one or more tasks running slowly in a stage will be re-launched
    * Datasets with UDF: you might hit a lot of garbage collection (you're converting records to java object)
* Slow aggregation: groupBy
    * increase # of partitions before aggregation
    * increase executor memory so that executor spill less to disk
    * if slow after aggregation, try to repartition so that data is more balanced
    * make sure you are filtering data and selecting column that's needed
    * ensure null values are represented correctly
* slow joins: it's also shuffle
    * different join ordering
    * partition prior to joining
* Slow IO
    * speculation can help (launching another task), but can result in duplicate data (for eventually consistent file system such as S3) https://stackoverflow.com/questions/46375631/setting-spark-speculation-in-spark-2-1-0-while-writing-to-s3
    * network bandwith
* Driver OutofMemory
    * broadcast join where data trying to broadcast is too big
* Executor OutofMemory
* Unexpected Nulls in Result?
    * data format change
* serialization errors ?
    * very uncommon with structured API

## Performance tuning
### Indirect performance enhancements
* Design choices:
    * Scala vs Java vs python vs R: when using UDF or RDD transformation (outside of Structured APIs), it's best to use Scala/java
    * DataFrames vs SQL vs Datasets vs RDD: 
        * DF, SQL, DS have same performance
        * if using UDF, performance hit when using python or R
        * if using RDD, use scala! when python runs RDD code it serializes a lot of data to and from python process
* Object Serialization in RDD:
    * when serializing custom data types: use Kyro https://github.com/EsotericSoftware/kryo
* Cluster Configurations
    * dynamic allocation: dynamically adjust the resources your application occupies based on workload
* Scheduling
    * `max-executor-cores": bound # of cores
    * FAIR scheduler for multiple jobs within an application
* Data at Rest
    * parquet: recommend for structured data storage. it's column-oriented storage
    * splittable, compressed file ~ few hundered MB
    * Table partitioning: partioning data correctly allows spark to skip many irrelevant files, storage manager such as Hive supports this
    * Bucketing: similar to partitioning, it will improve join
    * Number of files: don't have too many small files. you can always launch more tasks than there are input files (spark will split file for you)
    * Data localilty: HDFS
    * statistics collection: spark has cost-based query optimizer based on properties of input data. you'll need to collect/maintain statistics about your table and its column (not RDD or dataframe). this is still developing.
* Shuffle: consider external shuffle service. 
    * always use Kyro over java serailzation
* Memory pressure and garbage collection:
    * use structured API as much as possible
    * goal of GC tuning in spark: ensure only long-lived cached datasets are stored in Old generation and the Young generation is sufficiently sized to store all short-lived objects. (so that we can avoid full garbage collections)

### Direct performance enhancements
* Parallelism:
    * `spark.default.parallelism`: "Default number of partitions in RDDs returned by transformations like join, reduceByKey, and parallelize when not set by user."
    * `spark.sql.shuffle.partitions`: "the number of partitions that are used when shuffling data for joins or aggregations."
* Improved filtering: filter as early as you can
* Repartioning and Coalescing
    * `repartition` will cause shuffle, `coalesce` will merge partitions on same node into one partition
    * `repartition` prior to `join` or `cache` will help performance
    * custom partioning is possible
* UDF: avoid UDF, use Structured API
* Caching: if application reuse same datasets repeatedly
    * Idea:
        * DF, table, RDD will get stored in memory or disk
        * subsequent read will be fast
        * incurs serialization, deserialization, storage cost
    * real use cae: so that we can skip steps of reloading and cleansing data multiple times
        * builiding multiple statistical models from a single data set
        * iteratitive algoritm
    * Details: 
        * lazy operation
        * RDD vs Structured APIs: RDD caches physical data vs Structured API is done on physical plan
    * different storage level:
        * memory_only
        * memory_and_disk
        * memory_only_ser: RDD as serialized (will cost more to read)
        * memory_and_disk_ser
        * disk_only
        * memory_only_2, memory_and_disk_2,etc: to enable replication
        * off_heap
    * also `persist` option is available.
* Joins:
    * equi-joins are easiest for SPark to optimize
    * use broadcast variable for small dataframe
    * collect statistics on table before join
    * bucket your data appropriately to avoid large shuffle
* Aggregation: if you are using RDD, use `reduceByKey` over `groupByKey`
* Broadcast Variable: 