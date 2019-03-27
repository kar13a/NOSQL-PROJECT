# NOSQL-PROJECT

Spark shell
How it works: 

Spark shell is an interactive REPL which allows you to interact with spark while you type in your code. It only contains the basic functions included in spark core. Anything outside of spark core libraries such as TwitterUtils, KafkaStreams, Cassandra needs to be imported by providing packages (in the form of a jar) when you initiate spark shell. For our project Hector has created a jar that contains both twitter and cassandra packages for spark located in /home/cass/sandbox/spark_CassandraTwitter.jar.
Running your own twitter stream to Cassandra:
To run a twitter stream we will initiate spark shell with our current jar as following:


spark-shell --jars /home/cass/sandbox/spark_CassandraTwitter.jar

NOTE:  command :paste in the spark shell console helps you paste many things at once, might want to use it for some of these steps, such as pasting the streaming code. Once done pasting do ctrl-d to exit. 


Once the shell has started up, lets import the libraries we need to make our app work, copy and paste the content below in the shell:

import org.apache.spark.sql.SparkSession
import org.apache.spark.streaming.twitter.TwitterUtils
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.sql.cassandra._

Notice that we include the twitter, cassandra, and spark streaming libraries, which are not part of spark core. 



Twitter requires authentication to connect to the API, so for the following steps we will copy and paste the system configurations below in order to authenticate with twitter API once we start our streaming:

System.setProperty("twitter4j.oauth.consumerKey", "GTitgcUAyPH7GKS2zHNMn8awM")
System.setProperty("twitter4j.oauth.consumerSecret","U8nnCYfPfzlndbSbC3PEOJZqEysuCR87nBBj9u7oOglby6t6Vp")
System.setProperty("twitter4j.oauth.accessToken","1104940660568797185-UJw0xWZ8MLLC7Aj8lPIv8dArEy8rqc")
System.setProperty("twitter4j.oauth.accessTokenSecret","wnHT4qsNhjBQrCSEQ3z75FRU8n84W3JYxHINrab9g80NH")

Next we will initiate our entry point to our application by defining sparkContext and StreamingContext:

val sc = spark.sparkContext
spark.sparkContext.setLogLevel("Error")
val ssc = new StreamingContext(sc, Seconds(10))

Notice that our window of batch to process is 10 seconds, so for every batch the number of records is the amount streamed in 10 seconds. This can be changed. The Log level set to ERROR is helpful as it defaults to INFO and it can be very verbose. 

Now that we have all our preset features we can paste the code below which defines our streaming entry point, and transforms it to a dataframe that contains hashName (name) along with the count (count) of the frequency of each batch. The dataframe will show the first 100 records, before it persist to a cassandra keyspace of test01 and tablename tweets_test1. Note: Nothing will happen when you paste this code, the next step will initiate this. 

OLD
NOTE: the line  //  rddToDF.write.mode("append").cassandraFormat("tweets_test1","test01").save()  is commented out currently because I haven't created a table called tweets_test1 that contains the schema: name, count where name is the primary key. So it will not save to cassandra. I will fix this, if you do want to save, create the table ahead of time, and uncomment the line, and input your table info. 

NEw: I created a table for called tweets_test, and commented the line back in so now it will persist to a cassandra table every 10 seconds

// incoming stream
val stream = TwitterUtils.createStream(ssc, None)
// Read tags only
val tags = stream.flatMap { status => status.getHashtagEntities.map(_.getText)}
// Iterate through the status streams
tags
 //Count of hashtag per window batch of stream
 .countByValue()
 .foreachRDD { rdd =>
   // order by count, higher first
   val orderRDD = rdd.sortBy(kv => kv._2)
   //This block of code is to turn our stream into a dataframe foreach batch and persist it to Cassandra
   import spark.implicits._
   val rddToDF = orderRDD.toDF("name","count")
   rddToDF.show(100)
   // test01 in this case is the keyspace, and tweets_test1 is the tableName
   println("inserting into cassandra table")
  rddToDF.write.mode("append").cassandraFormat("tweets_test","test").save()
 }

Finally, to initialize the stream we will paste the content below, as soon as you paste the lines below, this will trigger a stream and you will be unable to interact with your spark shell, you will only be able to see the output. Noticed the second line below mentions that it is awaiting termination, so to stop the streaming you will use “ctrl-c” on your spark shell. 

Note: to paste the two lines below type in :paste first into the console, then paste, then do ctr-d to complete

ssc.start()             // Start the computation
ssc.awaitTermination()
