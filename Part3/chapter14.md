# Broadcast Variables
* a way to share an immutable value aroud the cluster (without function closure)
    * function closure can be inefficient (needs multiple deserialization - one per task)

```scala
val suppBroadcast = spark.sparkContext.broadcast(supplementalData)
// you access broadcast variable by broadcastvar.value
words.map(word => (word, suppBroadcast.value.getOrElse(word, 0)))
  .sortBy(wordPair => wordPair._2)
  .collect()
```

# Accumularors
* Propagating values back to driver
* Mutable, clusrer can safely update on a per-row basis
* accumulators can be updated only with an operation that is associative and commutative
* if accumulator updates are performed inside actions only, spark guarantee exactly once - restarted task will not update value
* if updates are performed also in transformation, each update can be applied more than once
    * accumulator updates are not guranteed to be executed when mad within a lazy transformation like map()
* You can define your own accumulator: extend `AccumulatorV2`

```scala
val accChina = new LongAccumulator
spark.sparkContext.register(accChina, "China")

//same as the first expression
val accChina2 = spark.sparkContext.longAccumulator("China")

// add to accumulator
def accChinaFunc(flight_row: Flight) = {
  val destination = flight_row.DEST_COUNTRY_NAME
  val origin = flight_row.ORIGIN_COUNTRY_NAME
  if (destination == "China") {
    accChina.add(flight_row.count.toLong)
  }
  if (origin == "China") {
    accChina.add(flight_row.count.toLong)
  }
}

// foreach is an action, run function against each row
flights.foreach(flight_row => accChinaFunc(flight_row))

// custom accumulator
class EvenAccumulator extends AccumulatorV2[BigInt, BigInt] {
  private var num:BigInt = 0
  def reset(): Unit = {
    this.num = 0
  }
  def add(intValue: BigInt): Unit = {
    if (intValue % 2 == 0) {
        this.num += intValue
    }
  }
  def merge(other: AccumulatorV2[BigInt,BigInt]): Unit = {
    this.num += other.value
  }
  def value():BigInt = {
    this.num
  }
  def copy(): AccumulatorV2[BigInt,BigInt] = {
    new EvenAccumulator
  }
  def isZero():Boolean = {
    this.num == 0
  }
}
```