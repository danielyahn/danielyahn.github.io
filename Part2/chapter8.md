# Join Types
* inner join, (left/right) outer join
* left semi join: keep rows in the left, and only the left, dataset where the key appears in the right dataset

```sql
SELECT name
FROM table_1 a
    LEFT SEMI JOIN table_2 b ON (a.name=b.name)

-- left semi join explained
SELECT name
FROM table_1 a
WHERE EXISTS(
    SELECT * FROM table_2 b WHERE (a.name=b.name))
```

* left anti join: keep rows in the left, and only the left, dataset where they do not appear in the right dataset

```sql
SELECT name
FROM table_1 a
    LEFT ANTI JOIN table_2 b ON (a.name=b.name)

-- left anti join explained
SELECT name
FROM table_1 a
WHERE NOT EXISTS(
    SELECT * FROM table_2 b WHERE (a.name=b.name))
```

* natural joins: perform a join by implictly matching the columns between the two datasets with the same names

* cross join (cartesian): match every row in the left dataset with every row in the right dataset
    * don't do it!! `spark.sql.crossJoin.enable=true` if you don't want any warning

```scala
// inner join
person.join(graduateProgram,  person.col("graduate_program") === graduateProgram.col("id")).show()

// outer join
person.join(graduateProgram,  person.col("graduate_program") === graduateProgram.col("id"), "outer").show()
```

# Challenges when using join
* Joins on Complex Type: make sure your join expression returns Boolean
* Duplicate column names: keys have same column name, but get returned as duplicate columns

```scala
//approach 1: different join expression (use string)
person.join(gradProgramDupe,"graduate_program").show()
//approach 2: drop column after the join
person.join(gradProgramDupe, joinExpr).drop(person.col("graduate_program"))
  .select("graduate_program").show()
// approach 3: rename column before join
val gradProgram3 = graduateProgram.withColumnRenamed("id", "grad_id")
val joinExpr = person.col("graduate_program") === gradProgram3.col("grad_id")
person.join(gradProgram3, joinExpr).show()
```

# How Spafk performs joins
* Communication strategies (node-to-node)
    * big table-to-big table: every node talks to every other node, and share data according to join keys
        * if your data is not partitioned well, it will cause a lot of network traffic
    * big table-to-small table (broadcast join): replicate small DataFrame onto every worker node 
        * join will be peformed on every individual node (CPU will be the bottleneck)
        * check the following syntax to force broadcast:

```scala
import org.apache.spark.sql.functions.broadcast
val joinExpr = person.col("graduate_program") === graduateProgram.col("id")
person.join(broadcast(graduateProgram), joinExpr).explain()
```

```sql
-- special comment syntax indicates broadcast join (although it's not enforced, optimizer can ignore them)
SELECT /*+ MAPJOIN(graduateProgram) */ * FROM person JOIN graduateProgram
  ON person.graduate_program = graduateProgram.id
```

    * little table-to-little table : let spark decide
* Computation strategies (per node)
