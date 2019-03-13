# MLlib
* package built on and included in Spark, that provides interfaces for gathering and cleansing data, feature engineering and feature selection, training and tuning large-scale supervised and unsupervised ML models, and using those models in production
* `org.apache.spark.spark.ml` interfaces with DataFrame, high level abstraction for building pipeline
* `org.apache.spark.mllib` interfaces with RDD, it's in maintenance mode (OLD!)

## High-level MLlib Concepts
* transformers: function that conver raw data in some way. primarily used in preprocessing and feature engineering
* estimator: algorithms to train model as well as some transformer (tf-idf, normalizer, etc..) 
* evaluator: measure performance
* pipeline: similar to scikit learn, bring transformers/estimator/evaluator together
* Vector: low level data type that consists of Dobules
    * can be sparse or dense

```scala
import org.apache.spark.ml.linalg.Vectors
val denseVec = Vectors.dense(1.0, 2.0, 3.0)
val size = 3
val idx = Array(1,2) // locations of non-zero elements in vector
val values = Array(2.0,3.0)
val sparseVec = Vectors.sparse(size, idx, values)
sparseVec.toDense
denseVec.toSparse
```

## MLlib in action

```scala
import org.apache.spark.ml.feature.RFormula
val supervised = new RFormula()
  .setFormula("lab ~ . + color:value1 + color:value2")
  // lab is a label column
  // . represents all columns but label column
  // two additional input column by interaction between color/value{1,2}

val fittedRF = supervised.fit(df) //transformer
val preparedDF = fittedRF.transform(df) 
preparedDF.show() //transformed output has original data / features / label column

//split train and test
val Array(train, test) = preparedDF.randomSplit(Array(0.7, 0.3))

import org.apache.spark.ml.classification.LogisticRegression
// create LR estimator
val lr = new LogisticRegression().setLabelCol("label").setFeaturesCol("features")

// train model (executed right away)
val fittedLR = lr.fit(train)

// show prediction
fittedLR.transform(train).select("label", "prediction").show()
```

```scala
// pipelining
val rForm = new RFormula()
val lr = new LogisticRegression().setLabelCol("label").setFeaturesCol("features")

import org.apache.spark.ml.Pipeline
val stages = Array(rForm, lr)
val pipeline = new Pipeline().setStages(stages)
```

```scala
//grid search
import org.apache.spark.ml.tuning.ParamGridBuilder
val params = new ParamGridBuilder()
  .addGrid(rForm.formula, Array(
    "lab ~ . + color:value1",
    "lab ~ . + color:value1 + color:value2"))
  .addGrid(lr.elasticNetParam, Array(0.0, 0.5, 1.0))
  .addGrid(lr.regParam, Array(0.1, 2.0))
  .build()

// create evaluator
import org.apache.spark.ml.evaluation.BinaryClassificationEvaluator
val evaluator = new BinaryClassificationEvaluator()
  .setMetricName("areaUnderROC")
  .setRawPredictionCol("prediction")
  .setLabelCol("label")

// split dataset and evalute models
import org.apache.spark.ml.tuning.TrainValidationSplit
val tvs = new TrainValidationSplit()
  .setTrainRatio(0.75) // also the default.
  .setEstimatorParamMaps(params)
  .setEstimator(pipeline)
  .setEvaluator(evaluator)

val tvsFitted = tvs.fit(train)
evaluator.evaluate(tvsFitted.transform(test))
```

```scala
//writing and reading model
tvsFitted.write.overwrite().save("/tmp/modelLocation")

import org.apache.spark.ml.tuning.TrainValidationSplitModel
// need to explicitly specify the type you saved as
val model = TrainValidationSplitModel.load("/tmp/modelLocation")
model.transform(test)
```