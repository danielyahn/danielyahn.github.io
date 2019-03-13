# Key Concepts
* Event time event: time is embedded in the data itself (data created)
* processing time: time stream-processing system receives data
* stateful processing: when you need to update intermediate information (starte) over longer periods of time.
    * e.g. aggregation on key
    * spark has a `state store`. it's in-memory, but checkpointed in storage
* arbitrary (custom) stateful processing:
    * e.g. user session information. start and stop time is very arbitrary. report errors in web app if 5+ events happen during user's session. deduplicate records over time.

# Windows on Event time
* Tumbling window: no overlap between windows
    * you can also aggregate with other key along with window
* Sliding windows

```scala
//tumbling window, stateful aggregation
import org.apache.spark.sql.functions.{window, col}
streaming.selectExpr("*","cast(cast(Creation_Time as double)/1000000000 as timestamp) as event_time")
  .groupBy(window(col("event_time"), "10 minutes")).count()
  .writeStream
  .queryName("events_per_window")
  .format("memory")
  .outputMode("complete")
  .start()

//sliding window example
import org.apache.spark.sql.functions.{window, col}
// window width is 10 minutes and interval is 5 minutes
withEventTime.groupBy(window(col("event_time"), "10 minutes", "5 minutes"))
  .count()
  .writeStream
  .queryName("events_per_window")
  .format("memory")
  .outputMode("complete")
  .start()
```

# Handling late data with watermarks
* Watermark: amount of time following given event after which we do not expect to see any more data from that time ("how late do I expect to see data?")
    * Complete mode: we will see intermediate result of watermarked window 
    * Append mode: information will not be available until window closes
    * if you don't specify watermark, spark will maintain data in memory forever

```scala
import org.apache.spark.sql.functions.{window, col}
withEventTime
  .withWatermark("event_time", "5 hours") //water mark for 5 hours
  .groupBy(window(col("event_time"), "10 minutes", "5 minutes")) //width: 10 minutes, interval: 5 minute
  .count()
  .writeStream
  .queryName("events_per_window")
  .format("memory")
  .outputMode("complete") 
  .start()
```

# Dropping duplicates in a stream
* drop duplicate message based on user specified keys
* need watermark, this is stateful processing

```scala
import org.apache.spark.sql.functions.expr

withEventTime
  .withWatermark("event_time", "5 seconds")
  .dropDuplicates("User", "event_time") //drop based on two columns
  .groupBy("User")
  .count()
  .writeStream
  .queryName("deduplicated")
  .format("memory")
  .outputMode("complete")
  .start()
```

# Arbitrary stateful processing
* Arbitrary window type. each group can be updated indepedently of any other group as well.
    * `mapGroupsWithState`: map over each group of data and generate data (one row for each group)
        * only supports `update` output mode
        * define the following:
            * 3 classes: input definition, state definition, optionally output definition
            * function to update the state based on a key, an iterator of events, previous state
            * time-out parameter
    * `flatMapGroupsWithState`: map over each group of data and generate data (one or more rows for each group)
        * only supports `append` and `update` output mode
        * define the following:
            * 3 classes: input definition, state definition, optionally output definition
            * function to update the state based on a key, an iterator of events, previous state
            * time-out parameter
        
* time-out: 
    * global parameter across all groups
    * you can set it based on both processing time and event time
        * processing time `GroupState.setTimeoutDuration(D ms)`
            * time-out will not occur before clock has advanced by D ms
            * if there is no data in the stream (for any group) for a while, there won't be any trigger and time-out
        * event time `GroupState.setTimeoutTimestamp`
            * need to specify watermark, data older than watermark is filtered out
            * time-out occurs when watermark advances beyond the set timestamp
            * you can set time-out for each group
    * check for time-out first before processing the values: `state.hasTimeOut` flag

```scala
//mapGroupsWithState example
// 1. input and state definition
case class InputRow(user:String, timestamp:java.sql.Timestamp, activity:String)
case class UserState(user:String,
  var activity:String,
  var start:java.sql.Timestamp,
  var end:java.sql.Timestamp)

// 2. helper function for next one
def updateUserStateWithEvent(state:UserState, input:InputRow):UserState = {
  if (Option(input.timestamp).isEmpty) {
    return state
  }
  if (state.activity == input.activity) {

    if (input.timestamp.after(state.end)) {
      state.end = input.timestamp
    }
    if (input.timestamp.before(state.start)) {
      state.start = input.timestamp
    }
  } else {
    if (input.timestamp.after(state.end)) {
      state.start = input.timestamp
      state.end = input.timestamp
      state.activity = input.activity
    }
  }

  state
}

// 3. function to update the state based on a key, an iterator of events, previous state
import org.apache.spark.sql.streaming.{GroupStateTimeout, OutputMode, GroupState}
def updateAcrossEvents(user:String,
  inputs: Iterator[InputRow],
  oldState: GroupState[UserState]):UserState = {
  var state:UserState = if (oldState.exists) oldState.get else UserState(user,
        "",
        new java.sql.Timestamp(6284160000000L),
        new java.sql.Timestamp(6284160L)
    )
  // we simply specify an old date that we can compare against and
  // immediately update based on the values in our data

  for (input <- inputs) {
    state = updateUserStateWithEvent(state, input)
    oldState.update(state)
  }
  state
}

// 4. start streaming job
import org.apache.spark.sql.streaming.GroupStateTimeout
withEventTime
  .selectExpr("User as user",
    "cast(Creation_Time/1000000000 as timestamp) as timestamp", "gt as activity")
  .as[InputRow]
  .groupByKey(_.user)
  .mapGroupsWithState(GroupStateTimeout.NoTimeout)(updateAcrossEvents)
  .writeStream
  .queryName("events_per_window")
  .format("memory")
  .outputMode("update")
  .start()
```