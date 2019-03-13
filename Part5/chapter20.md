#What is Stream processing?
* act of continously incorporating new data to compute a result
* **continuous application**: deliver streaming, batch and interactive jobs in one product
* use case:
    * notification, reporting
    * incremental ETL: it's critical that data is processed exactly once and in a fault-tolerant manner. also, ingest needs to be transactional (queries are not confused with partially written data)
    * update data to serve in real time: eg. Google analytic
    * real-time decision making: eg. credit card fraud detection
    * online machine learning: continously update ML models
* Advantages of stream processing
    * low latency (low throughput though)
    * efficient when updating result: e.g. moving average of past 24 hours
* Challenges of stream processing:
    * processing out-of-order data based on event time
    * maintaining large amounts of state
    * supporting high-data throughput
    * processing each event exactly once despite machine failures
    * handling load imbalance and stragglers
    * responding to events at low latency
    * joining with external data in other storage systems
    * determining how to update output sinks as new events arrive
    * writing data transactionally to output systems
    * updating your application's business logic at runtime

#Stream Processing Design Points
* Record-at-a-Time vs Declartive APIs
    * Record-at-a-time: early streaming system (e.g. Apache Storm). no support for maintaing states. required custom development.
    * declartive API: specify what to compute, but not how to compute or how to recover from failure
        * Kafka stream and Google Dataflow does similar
* Event Time vs Processing Time: how do you handle late-arrival? 
* Continous vs Micro-batch
    * continous: record at a time, low latency, low throughput. load balancing issue
    * micro-batch: high throughput, high latency, dynamic load balancing

# Spark's Streaming APIs
* DStream API
    * released in 2012
    * exactly-once semantic
    * Limitations: 
        * RDD only, no DataFrame or Datasets, so limits performance optimization
        * no event-time support
        * can only operate in micro-batch
* Structured streaming
    * built for all 5 (Scala, Java, Python, R, SQL)
    * native support for event time data
    * support for continous execution since Spark 2.3
    * increased reusability for other use cases (batch, interacive queries, etc.)
    * can use standard sinks usable by Spark SQL
* 

