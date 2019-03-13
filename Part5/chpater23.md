* Production-ready since 2.2

# Fault Tolerance and Checkpointing
* use checkpointing and write-ahead logs. if you restart application with the checkpoint, it will start automatically.

```scala
val query = streaming
  .writeStream
  .outputMode("complete")
  .option("checkpointLocation", "/some/location/")
  .queryName("test_stream")
  .format("memory")
  .start()
```

# Updating your applications
* Updating stream application code
    * You're allowed to change UDF as long as it keeps the same type signature
    * adding a new column should work
    * there are bigger changes that require checkpoint directory: such as new aggregation key, new query
* Updating spark version: forward compatible

# Sizing and Rescaling your application
* you can scale up and down (depending on resource manager configuration)
* Spark app configuration changes need application restart
* some sql setting (`spark.sql.shuffle.partitions) will require stream restart

# Metrics and monitoring
* basic Spark API: you need to build web UI
    * query status
    * recent progress
* use `StreamingQueryListener` to receive asynchronous updates from other systems