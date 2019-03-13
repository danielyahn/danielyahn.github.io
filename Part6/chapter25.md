* SQL Trnasformer

```scala
import org.apache.spark.ml.feature.SQLTransformer

val basicTransformation = new SQLTransformer()
  .setStatement("""
    SELECT sum(Quantity), count(*), CustomerID
    FROM __THIS__
    GROUP BY CustomerID
  """)

basicTransformation.transform(sales).show()
```
* VectorAssembler: concatenate all features into one vector
    * useful when you gather various transformations as a last step.
    * supports concatenation of Bool, Double, Vector columns

```scala
import org.apache.spark.ml.feature.VectorAssembler
val va = new VectorAssembler().setInputCols(Array("int1", "int2", "int3"))
va.transform(fakeIntDF).show()
```

* StringIndexer vs VectorIndexer:
    * stringIndexer: index single categorical column to double
    * vectorIndexer: index categorical vector to doubles
