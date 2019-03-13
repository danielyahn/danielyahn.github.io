# Aggregation Functions
* All aggregations are available as SQL functions.
* There are some gaps between available SQL functions and scala's library
* count:
    * don't get confused with the action we've been using such as `df.count()`
    * `count(*)` counts null values, whereas `count(<column-name>)` doesn't count null values.

```scala
import org.apache.spark.sql.functions.count
df.select(count("StockCode")).show() 

import org.apache.spark.sql.functions.countDistinct
df.select(countDistinct("StockCode")).show()

// approximate distinct count for performance improvement
import org.apache.spark.sql.functions.approx_count_distinct
df.select(approx_count_distinct("StockCode", 0.1)).show() 
```

* first and last: returns values from first row and last row

```scala
import org.apache.spark.sql.functions.{first, last}
df.select(first("StockCode"), last("StockCode")).show()
```

* min, max, sum, avg, var, stddev, skeness, kurtosis: column-wise aggregation

```scala
// distinct sum
import org.apache.spark.sql.functions.sumDistinct
df.select(sumDistinct("Quantity")).show()

// statistics
import org.apache.spark.sql.functions.{var_pop, stddev_pop}
import org.apache.spark.sql.functions.{var_samp, stddev_samp}
df.select(var_pop("Quantity"), var_samp("Quantity"),
  stddev_pop("Quantity"), stddev_samp("Quantity")).show()

//skewness: measure of asymmetry of the values around mean
//kurtosis: measure of tail of data
import org.apache.spark.sql.functions.{skewness, kurtosis}
df.select(skewness("Quantity"), kurtosis("Quantity")).show()
```

* covaraince, correlation: multiple column aggregation

```scala
import org.apache.spark.sql.functions.{corr, covar_pop, covar_samp}
df.select(corr("InvoiceNo", "Quantity"), covar_samp("InvoiceNo", "Quantity"),
    covar_pop("InvoiceNo", "Quantity")).show()
```

* Aggregating to Complex Types
    * aggregate to complex type, also aggregate on complex type

```scala
import org.apache.spark.sql.functions.{collect_set, collect_list}
df.agg(collect_set("Country"), collect_list("Country")).show()
```

# Grouping
* perform aggregation per group

```scala
df.groupBy("InvoiceNo", "CustomerId") //returns `RelationalGroupedDataSet`
   .count() //returns DataFrame
   .show()

// note that we're using sum/count function in aggregation function (instead of action)
df.groupBy("InvoiceNo").agg(
  count("Quantity").alias("quan"),
  sum("Quantity").alias("sum_quan")).show()

// you can do multiple aggregation by using map
df.groupBy("InvoiceNo").agg("Quantity"->"avg", "Quantity"->"stddev_pop").show()
```

# Window Functions
* unique aggregation on a specific window of data. Window determines which rows are grouped together
* with window function, each row can fall into one or more "frames"
* common use case: rolling average of some value where each row represents one day 
* 3 different types: ranking, analytic, aggregate functions

```scala
import org.apache.spark.sql.expressions.Window
import org.apache.spark.sql.functions.col
val windowSpec = Window
  .partitionBy("CustomerId", "date") //specify columns to use for grouping
  .orderBy(col("Quantity").desc) //ordering within partition
  .rowsBetween(Window.unboundedPreceding, Window.currentRow) //all previous rows up to current row

import org.apache.spark.sql.functions.max
val maxPurchaseQuantity = max(col("Quantity")).over(windowSpec)

import org.apache.spark.sql.functions.{dense_rank, rank}
val purchaseDenseRank = dense_rank().over(windowSpec)
val purchaseRank = rank().over(windowSpec)
```

# Grouping sets
* instead of aggregating on group of columns, let's aggregate across multiple groups
* For grouping sets, cubes, rollups: make sure to filter out null values
* Grouping Set is only available SQL, in dataframe use rollups and cubes intead.

# Rollup
* multidimensional aggregation, calculates various group-by combinations
* order matters for the list given in the rollup API (higher dimension comes first)

```scala
val rolledUpDF = dfNoNull.rollup("Date", "Country").agg(sum("Quantity"))
  .selectExpr("Date", "Country", "`sum(Quantity)` as total_quantity")
  .orderBy("Date")
// rolled up dataframe will include "quantity" aggregation for
// 1. each date and country combinatino
// 2. each date (across country)
// 3. everything
```

# Cube
* Similar to rollup, but does aggregation across all dimensions

```scala
dfNoNull.cube("Date", "Country").agg(sum(col("Quantity")))
  .select("Date", "Country", "sum(Quantity)").orderBy("Date").show()
```

# Grouping Metadata
* For cube and roll up outcome, this is easier way to filter (by aggregation level)

```scala
import org.apache.spark.sql.functions.{grouping_id, sum, expr}

dfNoNull.cube("customerId", "stockCode").agg(grouping_id(), sum("Quantity"))
.orderBy(expr("grouping_id()").desc)
.show()

/* 4 levels
(  Aggregate across )
customerId  stockCode   level
Yes         Yes         3
Yes         No          2
No          Yes         1
No          No          0
*/ 
```

# Pivot
* Convert a row into a column
* `val pivoted = dfWithDate.groupBy("date").pivot("Country").sum()`
* this dataframe will have:
    * columns for every combination of country and numeric variable. each column will contain aggregated value
    * a column speficying the date
* two roughly equivalent expressions

```scala
pivoted.where("date > '2011-12-05'").select("date" ,"`USA_sum(Quantity)`").show()
dfWithDate.where("Country='USA'").groupBy("date").sum("Quantity").where("date>'2011-12-05'").show()
```

# User-Defined Aggregation Functions
* You must inherit from `UserDefinedAggregateFunction` base class and implement
    * inputSchema (StructType): represents input arguments
    * bufferSchema (StructType): represents immediate UDAF results
    * dataType (DataType): represents return DataType
    * deterministic: Boolean value whether this UDAF is 
    * initialize: initialize aggregation buffer
    * update: how you should update internal buffer based on a given row
    * merge: how two aggregation buffers should merge
    * evaluate: generate final result of aggregation