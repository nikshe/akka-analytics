Akka Analytics
==============

Large-scale event processing with [Akka Persistence](http://doc.akka.io/docs/akka/2.3.4/scala/persistence.html) and [Apache Spark](http://spark.apache.org/). At the moment you can 

- batch-process events from an [Akka Persistence Cassandra](https://github.com/krasserm/akka-persistence-cassandra) journal as Spark `RDD`.
- stream-process events from an [Akka Persistence Kafka](https://github.com/krasserm/akka-persistence-kafka) journal as Spark Streaming `DStream`. 

This project is in **early preview** state. Development is currently suspended and will continue as soon as Spark upgrades to Akka 2.3.x (see also section [Dependencies](#dependencies)).

Dependencies
------------

    resolvers += "krasserm at bintray" at "http://dl.bintray.com/krasserm/maven"

    libraryDependencies ++= Seq(
      "com.github.krasserm" %% "akka-analytics-cassandra" % "0.1",
      "com.github.krasserm" %% "akka-analytics-kafka" % "0.1",
      "com.github.krasserm" %% "akka-persistence-kafka" % "0.2" intransitive(),
      "com.typesafe.akka" %% "akka-persistence-experimental" % "2.3.4" intransitive(),
      "org.apache.spark" %% "spark-core" % "1.0.1",
      "org.apache.spark" %% "spark-streaming" % "1.0.1",
      "org.apache.spark" %% "spark-streaming-kafka" % "1.0.1"
    )

Spark currently depends on Akka 2.2.x but Akka Persistence requires Akka 2.3.x which is binary incompatible with Akka 2.2.x. With an `intransitive()` inclusion of `akka-persistence-experimental` and `akka-persistence-kafka` all de-serializers and class definitions needed to process logged events are included but not Akka 2.3.x itself. The de-serializers of Akka Persistence 2.3.x work well with the serialization infrastructure of Akka 2.2.x which allows for some running examples but further development and production use is only possible after Spark upgrades to Akka 2.3.x (see also [SPARK-2707](https://issues.apache.org/jira/browse/SPARK-2707) and [SPARK-2805](https://issues.apache.org/jira/browse/SPARK-2805)).

Event batch processing
----------------------

With `akka-analytics-cassandra` you can expose and process events written by **all** persistent actors as [resilient distributed dataset](http://spark.apache.org/docs/latest/programming-guide.html#resilient-distributed-datasets-rdds) (`RDD`). It uses the [Spark Cassandra Connector](https://github.com/datastax/spark-cassandra-connector) to fetch data from the Cassandra journal. Here's a primitive example:

 ```scala
import org.apache.spark.rdd.RDD
import org.apache.spark.SparkConf
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._

import akka.analytics.cassandra._

val conf = new SparkConf()
  .setAppName("CassandraExample")
  .setMaster("local[4]")
  .set("spark.cassandra.connection.host", "127.0.0.1")
  .set("spark.cassandra.journal.keyspace", "akka") // optional, defaults to "akka"
  .set("spark.cassandra.journal.table", "messages") // optional, defaults to "messages"

val sc = new SparkContext(conf)

// expose journaled Akka Persistence events as RDD
val rdd: RDD[(JournalKey, Any)] = sc.eventTable().cache()

// and do some processing ... 
rdd.sortByKey().map(...).filter(...).collect().foreach(println)
```

The dataset generated by `eventTable()` is of type `RDD[(JournalKey, Any)]` where `Any` represents the persisted event (see also Akka Persistence [API](http://doc.akka.io/api/akka/2.3.4/#akka.persistence.package)) and `JournalKey` is defined as

```scala
package akka.analytics.cassandra

case class JournalKey(persistenceId: String, partition: Long, sequenceNr: Long)
```

Events for a given `persistenceId` are partitioned across nodes in the Cassandra cluster where the partition is represented by the `partition` field in the key. The `eventTable()` method returns an `RDD` in which events with the same `persistenceId` - `partition` combination (= cluster partition) are ordered by increasing `sequenceNr` but the ordering across cluster partitions is not defined. If needed the `RDD` can be sorted with `sortByKey()` by `persistenceId`, `partition` and `sequenceNr` in that order of significance. Btw, the default size of a cluster partition in the Cassandra journal is `50001` events (see [akka-persistence-cassandra](https://github.com/krasserm/akka-persistence-cassandra)).

Event stream processing
-----------------------

With `akka-analytics-kafka` you can expose and process events written by **all** persistent actors (more specific, from any [user-defined topic](https://github.com/krasserm/akka-persistence-kafka#user-defined-topics)) as [discretized stream](http://spark.apache.org/docs/latest/streaming-programming-guide.html#dstreams) (`DStream`). Here's a primitive example:

```scala
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.streaming.dstream.DStream

import akka.analytics.kafka._
import akka.persistence.kafka.Event

val sparkConf = new SparkConf()
  .setAppName("events-consumer")
  .setMaster("local[4]")

// read from user-defined events topic 
// with 2 threads (see also Kafka API) 
val topics = Map("events" -> 2)
val params = Map[String, String](
  "group.id" -> "events-consumer",
  "auto.commit.enable" -> "false",  
  "auto.offset.reset" -> "smallest",
  "zookeeper.connect" -> "localhost:2181",
  "zookeeper.connection.timeout.ms" -> "10000")

val ssc = new StreamingContext(sparkConf, Seconds(1))
val es: DStream[Event] = ssc.eventStream(params, topics)

es.foreachRDD(rdd => rdd.map(...).filter(...).collect().foreach(println))

ssc.start()
ssc.awaitTermination()
```

The stream generated by `eventStream(...)` is of type `DStream[Event]` where `Event` is defined in `akka-persistence-kafka` as

```scala
package akka.persistence.kafka

/**
 * Event published to user-defined topics.
 *
 * @param persistenceId Id of the persistent actor that generates event `data`.
 * @param sequenceNr Sequence number of the event.
 * @param data Event data generated by a persistent actor.
 */
case class Event(persistenceId: String, sequenceNr: Long, data: Any)
```

The stream of events (written by all persistent actors) is partially ordered i.e. events with the same `persistenceId` are ordered by `sequenceNr` whereas the ordering of events with different `persistenceId` is not defined. Details about Kafka consumer `params` are described [here](http://kafka.apache.org/documentation.html#consumerconfigs).

