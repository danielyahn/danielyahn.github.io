# Chapter 1
## Philosophy
1. Unified: unified platform for parallel data processing. `Structured APIs` in Spark 2.x is notable
2. Computing engine: many different storage option (S3, HDFS, Cassandra, Kafka, etc..)
3. Libraries: SparkSQL, MLlib, stream processing (spark streaming and structured streaming), GraphX, any many open-source external libraries

## History
1. AMPlab 2009
2. recognized need for API-based functional programming framework
3. 2014 - Spark 1 (Spark SQL), 2016 - Spark 2

# Chapter 2
## Basic Architecture
1. Application: one **driver** process and a set of **executor** processes.
    * Driver: maintains info about spark app, respond to user's program or input, analyze/distribute/schedule work across executors
    * Executor: executes work assigned by driver, report state of computation to driver
    * Note:
        * There are 3 core cluster managers: standalone, yarn, mesos.
        * you can configure how many executors fall on each cluster node
        * local mode: both driver and executor run on
        
## Language APIs: Scala, Java, Python, SQL (ANSI SQL 2003)
* Python/R: Spark session on JVM translates the code to JVM instructions.
* With structure APIs-alone, every language will have similar level of performance
* "Unstructured"="lower level", "structured"="high level" 

## Spark Session
* Spark shell command starts spark session
    * Spark Session is available as `spark`
* When using standalone application, you must create SparkSession object yourself
    * There's one-to-one correspondence between SparkSession and spark application

## DataFrames
* most common structured API, represents a table of data
* **schema**: list of columns and their types
* **partitions**: collection of rows that exist on one physical machine  

## transformation
* core data structure is immutable 
* to change data frame, you need to define set of transformations.
* transformations are not executed until you take **actions**
* two types:
    1. narrow (dependencies) tx: each input partition contributes to one output partition. spark can do **pipelinining** (applying multiple tx in-memory)
        * eg: where
    2. wide (dependenceis) tx: input partition contributes to many output partitions. **shuffle** is needed. will need to write to disk

## Lazy evaluation
* Spark can optimize entire data flow from end to end.
* eg: **predicate pushdown** - even if filter is applied the last, Spark will push the filter down automatically

## Actions
* triggers computation. your logical plan (with transformations) is compiled to physical plan and executed.
* 3 kinds of actions:
    1. view data in console
    2. collect data to native objects in the language
    3. write to output data source

## end-to-end example
* `spark.read` returns `DataFrameReader` object and you can set `option`s such as `inferSchema` or `header` for csv reader
* reading file is narrow tx, whereas sort is wide tx
* We can call `explain()` on any DataFrame object to view lineage and execution plan
* Spark outputs 200 shuffle partitions by default. You can reduce it by doing `spark.conf.set("spark.sql.shuffle.partitions","5")`

## Spark SQL
* you can register DataFrame as a table or view. 
    * eg: `flightData2015.createOrReplaceTempView("flight_data_2015")` 
* Then query it using pure SQL
    * `spark.sql`: run sql query against DF and return another DF
* Spark SQL and DF compiles down to same plan
* multi-transformation query example:
    * 7 steps: read, groupby, sum, rename, sort, limit, collect
    * these three expression are equivalent

    ```scala
    // The following 3 are equivalent
    ds.sort("sortcol")
    ds.sort($"sortcol")
    ds.sort($"sortcol".asc)
    ```

    1. Dataframe 

    ```scala
    import org.apache.spark.sql.functions.desc

    flightData2015 
    .groupBy ( "DEST_COUNTRY_NAME" ) 
    .sum ( "count" ) 
    .withColumnRenamed ( "sum(count)" , "destination_total" ) 
    .sort ( desc ( "destination_total" )) 
    .limit ( 5 ) 
    ```

    2. SQL

    ```scala
    val maxSql = spark.sql ( """ 
    SELECT DEST_COUNTRY_NAME, sum(count) as destination_total 
    FROM flight_data_2015 GROUP BY DEST_COUNTRY_NAME ORDER BY sum(count) DESC LIMIT 5 """ ) 
    ```