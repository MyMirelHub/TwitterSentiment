# Tutorial: Sentiment nalysis on streaming data using Azure Databricks

This will be an adaption of the microsoft tutorial with current twitter API constraints. The tutorial can be found at:

<https://docs.microsoft.com/en-us/azure/azure-databricks/databricks-sentiment-analysis-cognitive-services>

## Background Context

We know that Hadoop is designed from the beginning to be a batch processor at very large scale - but a batch processor. When you run someting on Hadoop you submit the job to yarn, and the job will be batched across the cluster.  

### Hadoop vs Spark - quick overview  

However many use cases have requried to deal with stream analytics,such as absorbing twitter feeds, data from IOT devices - stuff which is coming live or quasi-live for which Hadoop is unable to deal with due to its design nature of being a batch processor.

To overcome this the Apache foundation started another foundation called Databricks, and built another brick in the Hadoop ecosystem called spark which is designed to deal with stream data.

- Unlike Hadoop, Spark is a different component which does not require a filesystem.
- Spark itself is the in-memory distributed processing and computation component
- Hadoop is programmed in Scala, Spark in Java.
- Both languaes are binary compatibe and when you write a program in Scala it gets tranformed into the same code that Java gets tranformed into and both are executed by the Java virtual machine.
- If you need storage on Spark, it supports hdfs as as one of its possible file systems, but can also write on a more traditional file system, whereas in Hadoop you must have hdfs deployed.  

In this case I will take an example of Twitter sentiment analysis where we want to get data from Twitter Live, and based on the hastag store it. The second step is to process and apply an AI service on the Tweet and evaluate the sentiments positive negative or neutral. That's the second part of the machine learning Services, which is closer for example to AWS recognition.

### Key Components

This is an illuistration of the architecture used.
![alt text](https://docs.microsoft.com/en-us/azure/azure-databricks/media/databricks-sentiment-analysis-cognitive-services/databricks-cognitive-services-tutorial.png "D1")

- Azure Events Hubs: Comparable to AWS Kinesis, it's a data bus, made for rapidly recieving large amounts of stream data.
- Consumber Notebook: Will use the distributed spark computation engine and the storage- hdfs in this case, and  we use spark has the queue processor.
- Azure Cognigtitive Sercices: Managed API's, Within These cognitive Services there is one which is designed to do sentiment analysis, which is based on a deep Learning Network which has been preached trained to analyze English sentences.

## Steps

### Create an Event Hub Namespace

Property | Description
--- | --- | ---
*Name* | `your_name`
*pricing tier* | `standard`
*Resource group* | Create new: `SparkExample`
*location* | `default loaction`

### Within Namespace Create an Event Hub

Property | Description
--- | --- | ---
*Name* | `myhub`
*parition count* | `default`
*Message retention* | `default`
*Capture | `Off`

### Create a Spark Cluster(Databricks Service)

Property | Description
--- | --- | ---
*Workspace Name* | `yourname`
*Pricing Tier* | `Standard`
*Location | `West Europe`
*resource group* | `SparkExamle`

### Cluster Configuration

In Azure DataBricks, create a new cluster with the following configuration:

Property | Description
--- | --- | ---
*Cluster Name* | `mysparkcluster`
*Cluster mode* | `Standard`
*Databricks Runtime* | `4.2`
*worker Type* | `Default`

### Twitter API config

Follow Microsoft Tutorial in intital link

### Input Java code for spark engine processing instructions

#### Send Tweets to Event Hubs

In the SendTweetsToEventHub notebook, paste the following code, and replace the placeholder with values for your Event Hubs namespace and Twitter application that you created earlier. This notebook streams tweets with the keyword "Azure" into Event Hubs in real time.

```Java
import java.util._
import scala.collection.JavaConverters._
import com.microsoft.azure.eventhubs._
import java.util.concurrent._

val namespaceName = "<EVENT HUBS NAMESPACE>"
val eventHubName = "<EVENT HUB NAME>"
val sasKeyName = "<POLICY NAME>"
val sasKey = "<POLICY KEY>"
val connStr = new ConnectionStringBuilder()
            .setNamespaceName(namespaceName)
            .setEventHubName(eventHubName)
            .setSasKeyName(sasKeyName)
            .setSasKey(sasKey)

val pool = Executors.newFixedThreadPool(1)
val eventHubClient = EventHubClient.create(connStr.toString(), pool)

def sendEvent(message: String) = {
  val messageData = EventData.create(message.getBytes("UTF-8"))
  eventHubClient.get().send(messageData)
  System.out.println("Sent event: " + message + "\n")
}

import twitter4j._
import twitter4j.TwitterFactory
import twitter4j.Twitter
import twitter4j.conf.ConfigurationBuilder

// Twitter configuration!
// Replace values below with yours

val twitterConsumerKey = "<CONSUMER KEY>"
val twitterConsumerSecret = "<CONSUMER SECRET>"
val twitterOauthAccessToken = "<ACCESS TOKEN>"
val twitterOauthTokenSecret = "<TOKEN SECRET>"

val cb = new ConfigurationBuilder()
  cb.setDebugEnabled(true)
  .setOAuthConsumerKey(twitterConsumerKey)
  .setOAuthConsumerSecret(twitterConsumerSecret)
  .setOAuthAccessToken(twitterOauthAccessToken)
  .setOAuthAccessTokenSecret(twitterOauthTokenSecret)

val twitterFactory = new TwitterFactory(cb.build())
val twitter = twitterFactory.getInstance()

// Getting tweets with keyword "Azure" and sending them to the Event Hub in realtime!

val query = new Query(" #Azure ")
query.setCount(100)
query.lang("en")
var finished = false
while (!finished) {
  val result = twitter.search(query)
  val statuses = result.getTweets()
  var lowestStatusId = Long.MaxValue
  for (status <- statuses.asScala) {
    if(!status.isRetweet()){
      sendEvent(status.getText())
    }
    lowestStatusId = Math.min(status.getId(), lowestStatusId)
    Thread.sleep(2000)
  }
  query.setMaxId(lowestStatusId - 1)
}

// Closing connection to the Event Hub
 eventHubClient.get().close()
```

### Read tweets from Event Hubs

```scala
iimport org.apache.spark.eventhubs._

// Build connection string with the above information
val connectionString = ConnectionStringBuilder("<EVENT HUBS CONNECTION STRING>")
  .setEventHubName("<EVENT HUB NAME>")
  .build

val customEventhubParameters =
  EventHubsConf(connectionString)
  .setMaxEventsPerTrigger(5)

val incomingStream = spark.readStream.format("eventhubs").options(customEventhubParameters.toMap).load()

incomingStream.printSchema

// Sending the incoming stream into the console.
// Data comes in batches!
incomingStream.writeStream.outputMode("append").format("console").option("truncate", false).start().awaitTermination()
```

Covnerting the binary mode to string:

```scala
iimport org.apache.spark.sql.types._
import org.apache.spark.sql.functions._

// Event Hub message format is JSON and contains "body" field
// Body is binary, so we cast it to string to see the actual content of the message
val messages =
  incomingStream
  .withColumn("Offset", $"offset".cast(LongType))
  .withColumn("Time (readable)", $"enqueuedTime".cast(TimestampType))
  .withColumn("Timestamp", $"enqueuedTime".cast(LongType))
  .withColumn("Body", $"body".cast(StringType))
  .select("Offset", "Time (readable)", "Timestamp", "Body")

messages.printSchema

messages.writeStream.outputMode("append").format("console").option("truncate", false).start().awaitTermination()
```

### Run Sentiment Analysis on Tweets

In this section, you run sentiment analysis on the tweets received using the Twitter API. For this section, you add the code snippets to the same AnalyzeTweetsFromEventHub notebook.

Start by adding a new code cell in the notebook and paste the code snippet provided below. This code snippet defines data types for working with the Language and Sentiment API.

```scala
iimport org.apache.spark.sql.types._
import org.apache.spark.sql.functions._

// Event Hub message format is JSON and contains "body" field
// Body is binary, so we cast it to string to see the actual content of the message
val messages =
  incomingStream
  .withColumn("Offset", $"offset".cast(LongType))
  .withColumn("Time (readable)", $"enqueuedTime".cast(TimestampType))
  .withColumn("Timestamp", $"enqueuedTime".cast(LongType))
  .withColumn("Body", $"body".cast(StringType))
  .select("Offset", "Time (readable)", "Timestamp", "Body")

messages.printSchema

messages.writeStream.outputMode("append").format("console").option("truncate", false).start().awaitTermination()
```
