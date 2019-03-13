# Working with booleans
* equals: `===`
* not equals: =`!=` or `<>`
* interesting expressions:

```scala
// Apply bool column expression as filter
val priceFilter = col("UnitPrice") > 600
val descripFilter = col("Description").contains("POSTAGE")
df.where(col("StockCode").isin("DOT")).where(priceFilter.or(descripFilter))
  .show()

// Apply bool expression as column
val DOTCodeFilter = col("StockCode") === "DOT"
val priceFilter = col("UnitPrice") > 600
val descripFilter = col("Description").contains("POSTAGE")
df.withColumn("isExpensive", DOTCodeFilter.and(priceFilter.or(descripFilter)))
  .where("isExpensive")
  .select("unitPrice", "isExpensive").show(5)
```

equivalent to 
```sql
SELECT UnitPrice, (StockCode = 'DOT' AND
  (UnitPrice > 600 OR instr(Description, "POSTAGE") >= 1)) as isExpensive
FROM dfTable
WHERE (StockCode = 'DOT' AND
       (UnitPrice > 600 OR instr(Description, "POSTAGE") >= 1))
```

* null safety: `col("colname").eqNullSafe("hello")`

# Working with Numbers
* using SQL functions do basic math
    * pow
    * round
    * bround: round down

```scala
import org.apache.spark.sql.functions.{expr, pow}
val fabricatedQuantity = pow(col("Quantity") * col("UnitPrice"), 2) + 5
df.select(expr("CustomerId"), fabricatedQuantity.alias("realQuantity")).show(2)

// SQL like expression
df.selectExpr(
  "CustomerId",
  "(POWER((Quantity * UnitPrice), 2.0) + 5) as realQuantity").show(2)

import org.apache.spark.sql.functions.{round, bround}
df.select(round(col("UnitPrice"), 1).alias("rounded"), col("UnitPrice")).show(5)

import org.apache.spark.sql.functions.lit
df.select(round(lit("2.5")), bround(lit("2.5"))).show(2)
```

* using SQL functions do stats
    * corr
    * describe
    * stat packages: df.stat
        * such as quantile or crossTavulate

```scala
import org.apache.spark.sql.functions.{corr}
df.stat.corr("Quantity", "UnitPrice")
df.select(corr("Quantity", "UnitPrice")).show()

df.describe().show()
```

* using SQL function to create unique id: `df.select(monotonically_increasing_id()).show(2)

# Working with Strings
* see string functions section of the doc: lower, ltrim, rtrim, trim
* initcap: capitalize first character of word
* lpad, rpad:
* regex_replace: sql function

```scala
regexp_replace(e: Column, pattern: String, replacement: String)
regexp_replace(e: Column, pattern: Column, replacement: Column)
```

* translate: character level replacment

```scala
translate(src: Column, matchingString: String, replaceString: String)
```

* extract

```scala
//group id starts from 1 (0 returns all match)
regexp_extract(e: Column, exp: String, groupIdx: Int)
```

* contains

```scala
val containsBlack = col("Description").contains("BLACK")
val containsWhite = col("DESCRIPTION").contains("WHITE")
df.withColumn("hasSimpleColor", containsBlack.or(containsWhite))
  .where("hasSimpleColor")
  .select("Description").show(3, false)
```

* `var args`: coverting list of values into function arguments

```scala
val simpleColors = Seq("black", "white", "red", "green", "blue")
val selectedColumns = simpleColors.map(color => {
   col("Description").contains(color.toUpperCase).alias(s"is_$color")
}):+expr("*") // could also append this value
df.select(selectedColumns:_*).where(col("is_white").or(col("is_red")))
  .select("Description").show(3, false)
```

# Working with Dates and Timestamps
* before spark 2.1, spark uses machine's timezone if it's not passed in the original string.
* you can use `spark.conf.sessionLocalTimeZone` in the SQL configuration
* Spark's `TimestampType` supports only second-level precision. If you're working with milliseconds or microseconds, you'll need to operate as longs.

```scala
import org.apache.spark.sql.functions.{current_date, current_timestamp}
val dateDF = spark.range(10)
  .withColumn("today", current_date())
  .withColumn("now", current_timestamp())
dateDF.createOrReplaceTempView("dateTable")

//basic date math
import org.apache.spark.sql.functions.{date_add, date_sub}
dateDF.select(date_sub(col("today"), 5), date_add(col("today"), 5)).show(1)

import org.apache.spark.sql.functions.{datediff, months_between, to_date}
dateDF.withColumn("week_ago", date_sub(col("today"), 7))
  .select(datediff(col("week_ago"), col("today"))).show(1)
dateDF.select(
    to_date(lit("2016-01-01")).alias("start"),
    to_date(lit("2017-05-22")).alias("end"))
  .select(months_between(col("start"), col("end"))).show(1)

//parsing date from string
import org.apache.spark.sql.functions.{to_date, lit}
spark.range(5).withColumn("date", lit("2017-01-01"))
  .select(to_date(col("date"))).show(1)

// for failed parsing, Spark will silently return null
dateDF.select(to_date(lit("2016-20-12")),to_date(lit("2017-12-11"))).show(1)

// use can specify java date format
val dateFormat = "yyyy-dd-MM"
val cleanDateDF = spark.range(1).select(
    to_date(lit("2017-12-11"), dateFormat).alias("date"),
    to_date(lit("2017-20-12"), dateFormat).alias("date2"))

// you can also parse timestamp with format OR convert time to timestamp
import org.apache.spark.sql.functions.to_timestamp
cleanDateDF.select(to_timestamp(lit("2019-29-11"), dateFormat)).show(1)
cleanDateDF.select(to_timestamp(col("date"), dateFormat)).show()

// also convert timestamp to date
cleanDateDF.select(to_timestamp(col("date"), dateFormat).alias("ts")).select(to_date($"ts")).show()
```

# Working with Nulls in Data
* Schema definition's nullable type: doesn't stop you from putting null on the column. it is there to help SQL optimizer
* Two ways to handle null: (globally or per-column basis) 
  1. Explicitly drop nulls
  2. fill them with a value
* coalesce: join multiple columns and use first non-null value

```scala
import org.apache.spark.sql.functions.coalesce
df.select(coalesce(col("Description"), col("CustomerId"))).show()
```

* ifnull, nullif, nvl, nvl2: available in Spark SQL but I would use `when`

```scala
people.select(when(people("gender") === "male", 0)
  .when(people("gender") === "female", 1)
  .otherwise(2))
```

* drop: drop rows that contains null

```scala
df.na.drop() //drop rows with any null (default)
df.na.drop("any") //drop rows with any null (default)
df.na.drop("all") //drop rows with all null
df.na.drop("all", Seq("StockCode", "InvoiceNo")) //dropw rows if specified columns are all null
```

* fill: fill one or more columns with a set of values

```scala
//filling any null value
df.na.fill("All Null values become this string")
//filling any null value on selected columns
df.na.fill(5, Seq("StockCode", "InvoiceNo"))
//filling specific value on selected columns 
val fillColValues = Map("StockCode" -> 5, "Description" -> "No Value")
df.na.fill(fillColValues)
```

* replace: more flexible than replacing null `df.na.replace("Description", Map("" -> "UNKNOWN"))`
* ordering: `asc_nulls_first`, `asc_nulls_at`, etc.

# Working with Complex Types (Structs, Arrays, Maps)
* Structs: dataframe within dataframe

```scala
import org.apache.spark.sql.functions.struct
val complexDF = df.select(struct("Description", "InvoiceNo").alias("complex"))
complexDF.createOrReplaceTempView("complexDF")

complexDF.select("complex.Description")
complexDF.select(col("complex").getField("Description"))
```

* Arrays: putting data into array allow us several userful operations

```scala
import org.apache.spark.sql.functions.split
//making Description column to array
df.select(split(col("Description"), " ")).show(2)

//select first value from array
df.select(split(col("Description"), " ").alias("array_col"))
  .selectExpr("array_col[0]").show(2)

//array length
import org.apache.spark.sql.functions.size
df.select(size(split(col("Description"), " "))).show(2) 

//check array content (returns boolean)
import org.apache.spark.sql.functions.array_contains
df.select(array_contains(split(col("Description"), " "), "WHITE")).show(2)

//explode
import org.apache.spark.sql.functions.{split, explode}

df.withColumn("splitted", split(col("Description"), " "))
  .withColumn("exploded", explode(col("splitted")))
  .select("Description", "InvoiceNo", "exploded").show(2)
```

* Maps: key-value pairs of columns
```scala
import org.apache.spark.sql.functions.map
// Create a map-type column
df.select(map(col("Description"), col("InvoiceNo")).alias("complex_map")).show(2)

// select from map-type column
df.select(map(col("Description"), col("InvoiceNo")).alias("complex_map"))
  .selectExpr("complex_map['WHITE METAL LANTERN']").show(2)

// explode
df.select(map(col("Description"), col("InvoiceNo")).alias("complex_map"))
  .selectExpr("explode(complex_map)").show(2)
```

# Working with JSON
```scala
val jsonDF = spark.range(1).selectExpr("""
  '{"myJSONKey" : {"myJSONValue" : [1, 2, 3]}}' as jsonString""")

import org.apache.spark.sql.functions.{get_json_object, json_tuple}

//extract JSON object from string
jsonDF.select(
    get_json_object(col("jsonString"), "$.myJSONKey.myJSONValue[1]") as "column",
    json_tuple(col("jsonString"), "myJSONKey")).show(2)

// struct type to json
import org.apache.spark.sql.functions.to_json
df.selectExpr("(InvoiceNo, Description) as myStruct")
  .select(to_json(col("myStruct")))

// json to struct (requires schema)
import org.apache.spark.sql.functions.from_json
import org.apache.spark.sql.types._
val parseSchema = new StructType(Array(
  new StructField("InvoiceNo",StringType,true),
  new StructField("Description",StringType,true)))
df.selectExpr("(InvoiceNo, Description) as myStruct")
  .select(to_json(col("myStruct")).alias("newJSON"))
  .select(from_json(col("newJSON"), parseSchema), col("newJSON")).show(2)
```

# User-Defined Functions
* Custom transformations: take and return one or more columns
* UDF's are registered as temporary functions in SparkSession / Context. Functions are serialized and sent to workers
* Java/Scala: there can be performance issue if you create a lot of objects
* Python: expensive because data needs to be serialized to python
  * serialization itself is expensive
  * when data enters python, spark cannot manage memory of python workers (potential resource contention)
  * you can write UDF in scala and use in python
* it's best to define return types when defining functions
* When you want to optionally return a value from UDF, return Option type in Scala
* You can create UDF in Hive (both temporary and permanent)
