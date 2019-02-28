# Chapter 3

## Running Production Applications
* `spark-submit`: send your app code to cluster and launch it 
* specify resources and arguments

## Datasets: type-safe structured APIs
* Write statically typed code in Java and Scala. Datasets API is not available in python and R.
    * DataFrame is collection of Row objects.
    * Dataset API provides ability to assign Java/Scala class to records. Dataset is a collection of typed objects
* Type safety is important when multiple engineers are working together
* Syntax: `Dataset<T>` for Java and `Dataset[T]` for scala
    * use **case class** in scala
    * use JavaZBean pattern in java
    * it is important for spark to analyze type T and create schema 
* integrates well with other data structure
    * RDD to DataFrame
    * Dataframe to Datasets: as easy as `val flights = flightsDF.as[Flight]`
    * Apply SQL on Datasets
* Collect/take will return in type-safe manner

## Structured Streaming
* high level API for stream processing. Available since Spark 2.2
    * using Spark's structured API (no code change from your batch code)
* window function: helpful tool when aggregating by date
    ```scala
    import org.apache.spark.sql.functions.{window, column, desc, col} 
    staticDataFrame
    .selectExpr("CustomerId", "(UnitPrice * Quantity) as total_cost","InvoiceDate") .groupBy(col("CustomerId"),window(col("InvoiceDate"),"1 day"))
    .sum("total_cost").show(5) 
    ```

    * SQL equivalent:
    ```
    spark.sql("""select Customerid, sum(unitprice*quantity) as totalspent,
    year(invoicedate),month(invoicedate),day(invoicedate)
    from retail_data group by CustomerId,year(invoicedate),month(invoicedate),day(invoicedate)
    limit 5""").show
    ```
* use spark session's `readStream` method to read data from string
    ```scala
    val streamingDataFrame = spark.readStream.schema(staticSchema).option("maxFilesPerTrigger" , 1).format("csv").option("header", "true").load("/data/retail-data/by-day/*.csv")
    ```

    * returns dataframe object and you can check whether it's streaming by running `streamingDataFrame.isStreaming` (a method of datasets)

    * SQL expression is identical as batch job
*  There are various writeStream `format` as well as outputMode

## Machine Learning
* You can create a pipeline, fit (create vector indices), and then transform (to vector)
* Caching vector in memory would make training faster. 
* Class naming convention: 
    1. Model definition (where you specify hyperparmeter, etc..) has class name of *\<algorithm>* 
    2. Trained model has class name of *\<algorithm>Model* 

## Lower level API (RDD)
* RDD in python and scala is not equivalent