# Overview of Structure Spark Types
* For DataFrame `schema` maintains column name and its type
* DataFrame vs DataSets:
    * untyped vs typed
    * runtime check with schema vs compile time check with type class
    * Dataset only available for JVM language
    * DataFrame is Datasets of type `Row`
        * `Row` type is Spark's internal representation optimized for memory and computation. More efficient than JVM types
* columns can also have complex type such as array, map, or null value 
* Spark `Catalyst` maintains spark's own type info thru planning and processing. spark internally convert expression to its internal Catalyst representation
    * For each loanguage Spark type is mapped to language's type
* To work with the internal types
    
    ```scala
    import org.apache.spark.sql.types._
    ```

# Overview of Structured API Execution
1. Write DataFrame/Dataset/SQL
2. Spark converts it to logical plan
    - do not refer to executor/driver
    1. **user code** 
    2. **unresolved logical plan**: convert user's expression to `unresolved logical plan`
    3. **resolved logical plan**: with `catalog` resolve columns and tables in the `analyzer`. reject if not found.
    4. **optimized logical plan**: with Catalyst Optimizer, optimize by pushing down predicates and selections
        * Catalyst can be extended for domain-specific optimization
3. Spark transforms logical plan to physical plan.
    * choose best plan by comparing `cost model`'s
    * looks at physical attributes of a given table such as table/partition size
    * results in series of RDDs and transformations
4. Spark executes physical plan
    * runs code over RDD. Further optimization at runtime, generating native Java bytecode that can remove entire tasks or starges