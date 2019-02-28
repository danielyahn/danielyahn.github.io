# Schemas
* you can get schema-on-read. However, for production, it's better to explicitly define it.
* Schema consists of `StructType` which is collection of `StructField`
    * `StructField` represents a column. It has 4 attributes: name, type, boolean flag for missing value, meta data.
    * you can have complex type: StructType in StructType

# Columns and Expressions
* expressions: select, manipulate and remove columns from dataframe
    * columns are just expressions
    * columns and transformation of those columns comiple to same logical plan as parsed expression

    ```scala
    // these two are equivalent
    (((col("someCol") + 5) * 200) - 6) < col("otherCol")
    expr("(((someCol + 5) * 200) - 6) < otherCol")
    ```

* syntatic sugar:

    ```scala
    // equivalent expression to define columns
    col("myCol")
    column("myCol")
    expr("myCol")
    // equivalent expression to access column
    'myCol
    $"myCol"
    ```

* programatic access to columns: `df.columns`

# Records and Rows
* Row object is internally byte array. Byte array interface is not available to user.
* Row doesn't have schema. You can access value by positional index.

```scala
import org.apache.spark.sql.Row
val myRow = Row("Hello", null, 1, false)

myRow(0) // type Any
myRow(0).asInstanceOf[String] // String
myRow.getString(0) // String
myRow.getInt(2) // Int
```

# DataFrame transforation
* you can do the following:
    * add/remove rows or columns
    * transform row into column / column into row
    * change order of rows based on values in column

# select and selectExpr
* select method takes column names

```scala
//many different expressions to select the same column
import org.apache.spark.sql.functions.{expr, col, column}
df.select(
    df.col("DEST_COUNTRY_NAME"),
    col("DEST_COUNTRY_NAME"),
    column("DEST_COUNTRY_NAME"),
    'DEST_COUNTRY_NAME,
    $"DEST_COUNTRY_NAME",
    expr("DEST_COUNTRY_NAME"))
  .show(2)
```

* you can't mix up Column objects and string. Compilation error for: `df.select(col("DEST_COUNTRY_NAME"),"DEST_COUNTRY_NAME")`
* `selectExpr` is equivalent to `select(expr(...))` and is very useful for SQL like select
* You can add any non-aggregatin SQL statement as long as columns resolve.

```scala
df.selectExpr(
    "*", // include all original columns
    "(DEST_COUNTRY_NAME = ORIGIN_COUNTRY_NAME) as withinCountry")
  .show(2)
```
* also aggregation over entire dataframe by running

```scala
df.selectExpr("avg(count)", "count(distinct(DEST_COUNTRY_NAME))").show(2)
```

# Useful functions
## Literal
* pass explicit value to Spark. 

```scala
df.select(expr("*"),lit(1).as("one)).show(2)
//equivalent to
// 1. in SQL
// select *,1 as One from dfTable LIMIT2

// 2. withColumn expression
df.withColumn("numberOne",lit(1)).show(2)
```

## Renaming columns columns
```scala
// will add a new column that copies DEST_COUNTRY_NAME
df.withColumn("Destination", expr("DEST_COUNTRY_NAME")).columns
// will do in-place renaming
df.withColumnRenamed("DEST_COUNTRY_NAME", "dest").columns
```

## Handling reserved chracters in column name
* spaces, dashes are reserved characters
* in expression, you need to escape column name with backtick \`

```scala
import org.apache.spark.sql.functions.expr

// no need to escape when passing string
val dfWithLongColName = df.withColumn(
  "This Long Column-Name",
  expr("ORIGIN_COUNTRY_NAME"))

// need to escape with backtick when using SQL expression
dfWithLongColName.selectExpr(
    "`This Long Column-Name`",
    "`This Long Column-Name` as `new col`")
  .show(2)
```

## Removing columns
`df.drop("column_name")`

## Filtering Rows
```scala
//equivalents
df.filter(col("count") < 2).show(2)
df.where("count < 2").show(2)
```

* In scala, you must use `=!=` operator so that you compare string against evaluted column expression

```scala
df.where(col("ORIGIN_COUNTRY_NAME")=!= "Croatia")
```

## Distinct
`df.select("column_name").distinct()`

## Random sample/split
```scala
// withReplacement, fraction, seed
df.sample(false,0.5,3)
// returns array of data frames
df.randomSplit([0.25,0.75],seed)
``` 

## Concatenating and appending Rows
* DataFrames are immutable. so it creates a new DataFrame
* Two DF's need to have same schema
* But unions are based on column position, not schema. so make sure to have same column order

## Sorting
* `sort` and `orderBy`
* by default ascending, but can override

```scala
import org.apache.spark.sql.functions.{desc, asc}
df.orderBy(expr("count desc")).show(2)
df.orderBy(desc("count"), asc("DEST_COUNTRY_NAME")).show(2)
```

## Limit
* Sql's limit equivalent `df.limit(5).show()`

## Repartition and Coalese
* Repartition incurs full shuffle. 
    * useful when future number of partitions is greater than current # of partitions
    * You can repartition by column as well. Useful for when filtering by certain column often. `df.repartition(col("DEST_COUNTRY_NAME"))`
* Coalesce doesn't incur full shuffle. it'll combine partitions

## Taking data back to driver
```scala
val collectDF = df.limit(10)
collectDF.take(5) // take works with an Integer count
collectDF.show() // this prints it out nicely
collectDF.show(5, false) // doesn't truncate long string
collectDF.collect()
```

* `toLocalIterator` is interesting function. Allows you to iterate over entire dataset partition-by-partition
    