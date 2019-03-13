# When to Use Datasets
* Datasets only work for Scala and Java. 
* When using Dataset API, Spark converts Spark Row to object format you specified. There'll be conversion that slows down your oeprations.
* Then why do we use datasets?
    1. When operation you want to perform cannot be expressed using DataFrame manipuation
    2. When you need type-safety (tradeoff with performance)
        * middle ground: use dataset when you'd like to collect data and manipulate it by using single-node libraries? (do heavy ETDL with dataframe first)
    3. it's trivial to reuse scala's transformation logics with Datasets API due to similarity

# Creating Datsets
* you need to know and define schema ahead of time
* sscala uses case class (immutable, decomposable through pattern matching, allows for comparison based on structure)

```scala
case class Flight(DEST_COUNTRY_NAME: String,
                  ORIGIN_COUNTRY_NAME: String, count: BigInt)

val flightsDF = spark.read
  .parquet("data/flight-data/parquet/2010-summary.parquet/")
val flights = flightsDF.as[Flight]
```

# Transformation
* Datasets allow us to specify more complex and strongly typed transformations than we could perform on DataFrames aloen because we manipuate raw JVM types
* you can write non-UDF (not using Spark code) custom functions.

```scala
def originIsDestination(flight_row: Flight): Boolean = {
  return flight_row.ORIGIN_COUNTRY_NAME == flight_row.DEST_COUNTRY_NAME
}

// You can use same function in local code and spark code
flights.filter(flight_row => originIsDestination(flight_row)).first()
flights.collect().filter(flight_row => originIsDestination(flight_row))
```

* map: see wht it returns!

```scala
scala> val f = flights.first
f: Flight = Flight(United States,Romania,1)

scala> f.DEST_COUNTRY_NAME
res2: String = United States

scala> flights.map(_.DEST_COUNTRY_NAME)
res3: org.apache.spark.sql.Dataset[String] = [value: string]

scala> flights.map(_.DEST_COUNTRY_NAME).take(5)
res4: Array[String] = Array(United States, United States, United States, Egypt, Equatorial Guinea)
```

* JoinWith: you end up with two nested Datasets. Each column represents one Dataset.
    * regular join will lose JVM type information
    * JoinWith will keep your custom object type

* Grouping and aggregation
    * `groupBy, rollup, cube`: you will get DataFrame (lose type information)
    * use `groupByKey` instead:
        * you will write your own function: but spark can't optimize
        * you're basically adding a new column
    * `flatMapGroups`: you can also define custom map function on gathered groups
    * `reduceGroup`: you can define how groups should be reduced

```scala
flights.groupByKey(x => x.DEST_COUNTRY_NAME).count()


// COMMAND ----------

flights.groupByKey(x => x.DEST_COUNTRY_NAME).count().explain


// COMMAND ----------

def grpSum(countryName:String, values: Iterator[Flight]) = {
  values.dropWhile(_.count < 5).map(x => (countryName, x))
}
flights.groupByKey(x => x.DEST_COUNTRY_NAME).flatMapGroups(grpSum).show(5)

def grpSum2(f:Flight):Integer = {
  1
}
flights.groupByKey(x => x.DEST_COUNTRY_NAME).mapValues(grpSum2).count().take(5)

def sum2(left:Flight, right:Flight) = {
  Flight(left.DEST_COUNTRY_NAME, null, left.count + right.count)
}
flights.groupByKey(x => x.DEST_COUNTRY_NAME).reduceGroups((l, r) => sum2(l, r))
  .take(5)
```

