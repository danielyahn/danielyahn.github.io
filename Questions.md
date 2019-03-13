# Questions:
1. List of options available to DataFrame reader?
    1. Such as sample rate for infer schema
2. Column expression syntax? string, $\<colname>, '\<colname> etc?
3. How does dataframe sql like function takes function as parameters?
4. Why is datasets under sql package
5. list of important spark configurations such as `spark.sql.shuffle.partitions`
6. How come most of classes are under `org.apache.spark.sql.*`?
7. lpad, rpad don't understand
8. Chapter 18 p312-314
9. What's the use case that requires RDD transformation or UDF's (which are both outside of structured API)
10. GC collection tuning 324
11. difference: `spark.default.parallelism` vs `spark.sql.shuffle.partitions`
12. collect statistics on table before join. how do you do that?
13. Can't find Read mode for scala doc Pg155
14. json_tuple p110
15. window function: p127-129, i understand common use case moving average, but not the book's example
16. samplebykey: 227-228

# Completed (23):
* Part I: 1, 2, 3
* Part II: 4, 5, 6, 7, 8, 9, 10, 11
* Part III: 12, 13, 14
* Part IV: 15, 16, 17, 18, 19
* Part V: 20, 21, 22, 23
* Part VI:

# Todo (8):
* Part VI: 24, 25, 26, 27, 28, 29, 30, 31

# Plan
* 8: Review 2 a day (Tue, Wed, Thur, Fri) 
* 12: Review 4+ a day (Sat, Sun, Mon)
* Evaluate where I am on Friday night

# Typo:
P102: not having a null time => not having a null type
P105: ordering - wrong header 
P155: arcquet format => parquet format
p168: missing instruction on how to load sqlLite jar
p200: missing import spark.implicits._
```
error: Unable to find encoder for type Flight. An implicit Encoder[Flight] is needed to store Flight instances in a Dataset. Primitive types (Int, String, etc) and Product types (case classes) are supported by importing spark.implicits._  Support for serializing other types will be added in future releases.
       val flights = flightsDF.as[Flight]
```