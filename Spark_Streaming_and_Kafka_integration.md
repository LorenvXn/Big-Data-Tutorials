<b> Used environment</b>

![ScreenShot](https://github.com/Satanette/test/blob/master/ss1.png)

```
# more /opt/mapr/MapRBuildVersion
5.2.1.42646.GA
This MapR version is already shipped with Zookeeper, Kafka and Spark (among others)
```

<b> Prepare Kafka environment </b> 

1) From another terminal, start Zookeeper
```
#  /opt/mapr/zookeeper/zookeeper-3.4.5/bin/zkServer.sh start /opt/mapr/kafka/kafka-0.9.0/config/zookeeper.properties &
[1] 47384
# JMX enabled by default
Using config: /opt/mapr/kafka/kafka-0.9.0/config/zookeeper.properties
Starting zookeeper ... STARTED
```
2) Open another terminal, and start Kafka server ( the broker that is…)
```
# /opt/mapr/kafka/kafka/kafka-0.9.0/bin/kafka-server-start.sh config/server.properties
```
3) For this example, topic fast-messages will be created and used:

```
# /opt/mapr/kafka/kafka/kafka-0.9.0/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 \
--partitions 1 --topic   fast-messages
```
Created topic "fast-messages".
..and find the group-id

```
# /opt/mapr/kafka/kafka-0.9.0/bin/zookeeper-shell.sh localhost ls "/consumers"
Connecting to localhost

WATCHER::
[console-consumer-33373, console-consumer-6246, console-consumer-91714, console-consumer-73719, console-consumer-89610, console-consumer-
3890, console-consumer-75371, console-consumer-3904, console-consumer-72157, console-consumer-3979, console-consumer-52304, console-
consumer-49562, console-consumer-39890, console-consumer-35136]

 
# /opt/mapr/kafka/kafka-0.9.0/bin/zookeeper-shell.sh localhost ls "/consumers/console-consumer-6246/offsets"
Connecting to localhost

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[fast-messages]
#
```
…so the group ID we will be using for our code is <b> console-consumer-6246 </b> 

4) Get to the spark-shell:

```
#/opt/mapr/spark/spark-2.1.0/bin/spark-shell
Spark context Web UI available at http://84.40.63.182:4040
Spark context available as 'sc' (master = local[*], app id = local-1494925705762).
Spark session available as 'spark'.
Welcome to
    ____              __
   / __/__  ___ _____/ /__
  _\ \/ _ \/ _ `/ __/  '_/
 /___/ .__/\_,_/_/ /_/\_\   version 2.1.0-mapr-1703
    /_/

Using Scala version 2.11.8 (OpenJDK 64-Bit Server VM, Java 1.8.0_131)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
…now add the below code: Scala code for integrating Spark with Kafka Consumer
```

//did the below code to be run strictly from spark-shell … it makes life easier!//

```
import _root_.kafka.serializer.DefaultDecoder
import _root_.kafka.serializer.StringDecoder
import org.apache.spark.storage.StorageLevel
import org.apache.spark.streaming._
import org.apache.spark.streaming.kafka09.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka09.ConsumerStrategies.Subscribe
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.kafka09._
import org.apache.spark.streaming.kafka09.{ConsumerStrategies, KafkaUtils, LocationStrategies}

import org.apache.kafka.clients.consumer.ConsumerConfig


sc.setLogLevel("ERROR")

val ssc = new StreamingContext(sc, Seconds(5))
val brokers = "localhost:9092"
val groupId="console-consumer-6246"
val offsetReset="earliest"
val pollTimeout ="1000"

val topics1 = Array("fast-messages")

  val kafkaParams = Map[String, String](
      ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG -> brokers,
     ConsumerConfig.GROUP_ID_CONFIG -> groupId,
      ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG ->
        "org.apache.kafka.common.serialization.StringDeserializer",
      ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG ->
        "org.apache.kafka.common.serialization.StringDeserializer",
      ConsumerConfig.AUTO_OFFSET_RESET_CONFIG -> offsetReset,
      ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG -> "false",
      "spark.kafka.poll.time" -> pollTimeout)
  
val consumerStrategy = ConsumerStrategies.Subscribe[String, String](topics1, kafkaParams)


val messages = KafkaUtils.createDirectStream[String, String](ssc, LocationStrategies.PreferConsistent, consumerStrategy)


   val lines = messages.map(_.value())
    val words = lines.flatMap(_.split(" "))
    val wordCounts = words.map(x => (x, 1L)).reduceByKey(_ + _)
    wordCounts.print()

ssc.start()

```
<i>This piece of code will make Spark wait for messages from Kafka-producer</i>

5) Start the Kafka Producer:
```
# /opt/mapr/kafka/kafka-0.9.0/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic  fast-messages
```
6) Send a few messages and see if they are written in Spark Shell

![ScreenShot](https://github.com/Satanette/test/blob/master/ss2.png)

Nice! It's working!

Time to implement the HDFS part of the code, and send bigger messages/files.

a) create a new folder, where the output will be received (/tmp/backup_streaming in our case):
```
hadoop fs -mkdir /tmp/backup_streaming
```
b) In the above code, right after creating the stream, add the following lines:
```
 val now: Long = System.currentTimeMillis 
 val hdfsdir = "/tmp/kafka_testing/"
 val lines = messages.map(_.value())

lines.foreachRDD(rdd => {
     if (rdd.count() > 0) {
 rdd.saveAsTextFile(hdfsdir+"file_"+now.toString())
 }
 })
 ```
 
This will save the RDD as a text file, under the mentioned folder, with the date of when this process is implemented.

So, your code for transfering streams into HDFS will be looking as below:

```import _root_.kafka.serializer.DefaultDecoder
import _root_.kafka.serializer.StringDecoder
import org.apache.spark.storage.StorageLevel
import org.apache.spark.streaming._
import org.apache.spark.streaming.kafka09.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka09.ConsumerStrategies.Subscribe
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.kafka09._
import org.apache.spark.streaming.kafka09.{ConsumerStrategies, KafkaUtils, LocationStrategies}
import org.apache.kafka.clients.consumer.ConsumerConfig


sc.setLogLevel("ERROR")

val ssc = new StreamingContext(sc, Seconds(5))
val brokers = "localhost:9092"
val groupId="console-consumer-6246"
val offsetReset="earliest"
val pollTimeout ="1000"

val topics1 = Array("fast-messages")

val kafkaParams = Map[String, String](
    ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG -> brokers,
   ConsumerConfig.GROUP_ID_CONFIG -> groupId,
    ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG ->
      "org.apache.kafka.common.serialization.StringDeserializer",
    ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG ->
      "org.apache.kafka.common.serialization.StringDeserializer",
    ConsumerConfig.AUTO_OFFSET_RESET_CONFIG -> offsetReset,
    ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG -> "false",
    "spark.kafka.poll.time" -> pollTimeout)

val consumerStrategy = ConsumerStrategies.Subscribe[String, String](topics1, kafkaParams)

val messages = KafkaUtils.createDirectStream[String, String](ssc, LocationStrategies.PreferConsistent, consumerStrategy)

//  the change starts here
 val now: Long = System.currentTimeMillis
 
 val hdfsdir = "/tmp/backup_streaming/"     //do mind the last slash 

 val lines = messages.map(_.value())

lines.foreachRDD(rdd => {
     if (rdd.count() > 0) {
 rdd.saveAsTextFile(hdfsdir+"file_"+now.toString())
 }
 })

 //and ends up here  
ssc.start()
```

Let's send a csv file with the Kafka Producer

Download a csv (use the wget command).

<i> I have downloaded the file from this link: https://github.com/cloudera/hue/blob/master/apps/beeswax/data/sample_07.csv </i>
```
# /opt/mapr/kafka/kafka-0.9.0/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic  fast-messages < /home/sample_07.csv
# echo $?
  0
#
```
And now, let's check under our folder if files were created:

![ScreenShot](https://github.com/Satanette/test/blob/master/ss4.png)

And finally, let's check what it contains:

merely a snip – For certainty, check the .csv file I have downloaded.

![ScreenShot](https://github.com/Satanette/test/blob/master/ss3.png)

And that's it!

Few tricks - you can easily check for any code issues within a Zeppelin.

We will be using Zeppelin next time, on how to integrate it with MapR - when real time analysis example will be presented 
(with machine learning algorithms & prediction graphs)

This time, if you check the above scheme, we have implemented paths (A), (B) and (D). 

(C) in the next tutorial, for [Part II](https://github.com/Satanette/Big-Data-Tutorials/blob/master/Real%20Time_Analysis_and_Prediction_with_Zeppelin.md)

Useful links:

<i>http://kafka.apache.org/documentation.html#quickstart </br>
https://kafka.apache.org/082/documentation.html </br>
http://spark.apache.org/docs/latest/streaming-programming-guide.html </br>
http://spark.apache.org/docs/latest/quick-start.html</i>
