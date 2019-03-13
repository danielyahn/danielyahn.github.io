# Relationship to Hive
* Hive was the defacto big data SQL access layer
* Since 2.0, Spark supports both ANSI-SQL and HiveQL queries
* Spark SQL can connect to Hive metastores
    * maintains table information for use across sessions
    * reduces file listing when accessing
* to connect to Hive metastore:
    * `spark.sql.hive.metastore.version`
    * `spark.sql.hive.metastore.jars`

# Spark's SQL interfaces
* Spark SQL CLI: command line tool to run basic Spark SQL querieis in local mode
    * can't communicate with the Thrift JDBC server
* SparkSession object's sql method: `spark.sql("select 1+1").show`
    * returns dataframe
    * also if you use createOreReplaceTempView on dataframe, you can use SQL
* SparkSQL Thrift JDBC/ODBC Server
    * JDBC interface so that you can execute Spark SQL remotely
    * it ported Apache Hive's HiveServer2
    * it's a standalone  application that can be started with script
    * SQL queries in the Thrift server share the same SparkContext
    * beeline is a commandline tool to connect to Spark Thrift Server through JDBC interface

# Catalog
* abstraction for storage of metadata about table, databases, functions, views
* programmatic interface to Spark SQL: createTable, getTable, getDatabase, etc..

# Tables
* in order to do anything with Spark SQL, you need to define table. THey are logically equivalent to Dataframe
    * scope of table: database
    * scope of dataframe: programming language
* In Spark 2.x, tables always contain data. if you drop table, there's risk of losing data
* Spark-managed table:
    * when you define a table from files on disk, the table is *unmanaged*
    * when you use `saveAsTable` on Dataframe, you are creating a *managed* table. Spark will save metadata as well.
    * it writes to the default Hive location (`spark.sql.warehouse.dir`)

## Creating tables
* https://docs.databricks.com/spark/latest/spark-sql/language-manual/create-table.html
* `USING` clause specifies data format. if you don't specify a format, it'll create a Hive-compatible table (it'll be slower though)

```sql
CREATE TABLE flights (
  DEST_COUNTRY_NAME STRING, ORIGIN_COUNTRY_NAME STRING, count LONG)
USING JSON OPTIONS (path '/data/flight-data/json/2015-summary.json')
```

* `PARTITIONED BY`: partition by column
## Creating external table: 
* Unmanaged table: files are not managed by Spark, but table's metadata are

```sql
-- Creating external table
CREATE EXTERNAL TABLE hive_flights (
  DEST_COUNTRY_NAME STRING, ORIGIN_COUNTRY_NAME STRING, count LONG)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' LOCATION 'data/flight-data-hive/'

-- Inserting from another table
CREATE EXTERNAL TABLE hive_flights_2
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/data/flight-data-hive/' AS SELECT * FROM flights
```
## Inserting into tables
* you can optionally provide a partition specification

```sql
INSERT INTO partitioned_flights
  PARTITION (DEST_COUNTRY_NAME="UNITED STATES")
  SELECT count, ORIGIN_COUNTRY_NAME FROM flights
  WHERE DEST_COUNTRY_NAME='UNITED STATES' LIMIT 12
```

## Describing Table metadata

```sql
-- Describe columns
DESCRIBE TABLE flights_csv
-- Show partition information
SHOW PARTITIONS partitioned_flights
```

## Refreshing Table metadata

```sql
-- refreshes cached entries (especially files) associated with table
REFRESH table partitioned_flights
-- refreshes partitions maintained in the catalog
MSCK REPAIR TABLE partitioned_flights
```

## Dropping Tables
* managed table: data will be removed
* unmanged table: no data will be removed.

## Cache

```sql
CACHE TABLE flights
```

# Views
* View specifies a set of transformations on top of an existing table, saved query plans for reusability
    * query plans/transformations get executed at query time (same execution plan as dataframe)
* you can set scope, global, database or per session
    * Temp View: avaialble for session only
    * Global Temp View: available for all spark app (regardless of database), but removed at the end of session
* you can drop view, create Or Replace view

# Databases
* If you don't define one, spark will use default one
* Any SQL statement (including DataFrame commands) will execute within context of a database
* Useful commands

```sql
SHOW DATABASES
CREATE DATABASE some_db
USE some_db
SHOW tables
```

# Advanced topics
* Complex types: Spark SQL supports structs, lists
    * struct: set of columns (similar to map)
    * list

```sql
--list as ggregator
SELECT DEST_COUNTRY_NAME as new_name, collect_list(count) as flight_counts,
  collect_set(ORIGIN_COUNTRY_NAME) as origin_set
FROM flights GROUP BY DEST_COUNTRY_NAME

--list in column
SELECT DEST_COUNTRY_NAME, ARRAY(1, 2, 3) FROM flights

-- opposite of collect_list: explode
-- DEST_COUNTRY_NAME will be duplicated for every value in array collected_counts
SELECT explode(collected_counts), DEST_COUNTRY_NAME FROM flights_agg
```

# Functions
* `show functions like "collect*"`
* you can register user function on spark session using language of your choice

# Subqueries
* Two types:
    1. Correlated subqueries: use information from outer scope of the query
    2. Uncorrelated subqueries: no information from the outer scope
* also supports predicat subqueries: filter based on values

```sql
--uncorelated
SELECT * FROM flights
WHERE origin_country_name IN (SELECT dest_country_name FROM flights
      GROUP BY dest_country_name ORDER BY sum(count) DESC LIMIT 5)
-- correlated
-- pretty cool! where exists returns true when subquery returns something
-- this is query to filter flights that have flights both way
SELECT * FROM flights f1
WHERE EXISTS (SELECT 1 FROM flights f2
            WHERE f1.dest_country_name = f2.origin_country_name)
AND EXISTS (SELECT 1 FROM flights f2
            WHERE f2.dest_country_name = f1.origin_country_name)

-- uncorrelated scalar queries
SELECT *, (SELECT max(count) FROM flights) AS maximum FROM flights
```

# Miscellaenous Features
* You Set configuration by using SET: `SET spark.sql.shuffle.partitions=20`
