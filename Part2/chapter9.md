* 6 core data sources: CSV, JSON, Parquet, ORC, JDBC, pain text AND many community-created data source
# Structure of the Data Sources API
* `DataFrameReader` for reading data (accessible in `SparkSession`)

```scala
//spark is SparkSession object
spark.format(...).option("key","value").schema(...).load()
```

* `DataFrameWriter` for writing data (accessible in `Dataframe`). 
    * Format: parquet by default
    * one file per partition. for file-based data source, you can control layout of destination files by doing: `partitionBy`, `bucketBy`, `sortBy`
    * save mode: overwrite, append, ignore, error https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.DataFrameWriter

```scala
//df is dataframe object
df.write.format(...).option(...,...).save()
```

# notable options:
* csv: inferSchema, sep, header, mode (FAILFAST, permissive, dropmalformed)
* json: 
    * multiLine: by default false, because we like line-delimited JSON files
* parquet: column-oriented data store (can read by column)
    * supports complex type such as array, map, struct
    * `spark.read.format("parquet")` will use schema stored on parquet files
    * `merge Schema`: incrementally add columns to newly written parquet files
* ORC files: self-describing, type-aware columnar file
    * similar to parquet, optimized for Hive. 

# SQL Datbases
* you need to specify JDBC jar and include in classpath
* Notable options:
    * partitionColumn, lowerBound, upperBound, numPartitions: for read
    * numPartition: for read and write, controls # of concurrent connections
        * when write, if you have more partitions, then spark will coalesce before writing
    * fetchsize: # of rows to fetch per round trip (for read)
    * batchsize: # of rows to insert per round trip (for write)
    * truncate: false by default (if true, it will drop table and recreate)
    * createTableOptions, createTableColumn: table configurations on SQL side when writing
* Query push down: when reading, spark runs filter on DB side first
    * you can also use select statement as table

    ```scala
    val pushdownQuery = """(SELECT DISTINCT(DEST_COUNTRY_NAME) FROM flight_info)
  AS flight_info"""
    val dbDataFrame = spark.read.format("jdbc")
    .option("url", url).option("dbtable", pushdownQuery).option("driver",  driver)
    .load()
    ```

* custom partitions by where clause:
    * array of predicates: each predicate forms a partition
    
    ```scala
    val props = new java.util.Properties
    props.setProperty("driver", "org.sqlite.JDBC")
    val predicates = Array(
    "DEST_COUNTRY_NAME = 'Sweden' OR ORIGIN_COUNTRY_NAME = 'Sweden'",
    "DEST_COUNTRY_NAME = 'Anguilla' OR ORIGIN_COUNTRY_NAME = 'Anguilla'")
    spark.read.jdbc(url, tablename, predicates, props).show()
    spark.read.jdbc(url, tablename, predicates, props).rdd.getNumPartitions
    ```

* partitoning based on a sliding window (on a column)
    
    ```scala
    val colName = "count"
    val lowerBound = 0L
    val upperBound = 348113L // this is the max count in our database
    val numPartitions = 10

    spark.read.jdbc(url,tablename,colName,lowerBound,upperBound,numPartitions,props)
    .count()
    ```
* Write mode: overwrite vs append

# Text Files
* Each line in the file becomes a record in the Dataframe
* When writing text file, ensure that you only have one string column
* if you partition by column, it will be separate file when writing

# Advanced I/O Concepts
* Splittable file types and compression: supported by certain formats only
    * avoid reading entire file
    * On HDFS, split a file if file spans over multiple blocks
    * not all compression schemes are splittable
    * recommend parquet with gzip
* Reading data in parallel
    * multiple executors can read different files at the same time
    * in general, each file will become a partition in your data frame
* Writing data in parallel
    * partition: control what data is stored (and where)
        * column name will form a separate folder
        * optimization for reads
        * date is common option for partition
    * bucketing: instead of grouping by a column's unique values, bucket by range
        * suppored only for Spark-managed tables
* Managing file size:
    * important for read, we don't want many small files
    * not only # of partitions, you can also use `maxRecordsPerFile` option