# Twitter-Data-Analysis-by-Spark-Streaming-on-AWS
This project shows how to use spark streaming capabilities to analyze twitter data directly from Twitter’s sample stream.
import com.google.gson.Gson
import org.apache.spark.streaming.twitter.TwitterUtils
import org.apache.spark.streaming._
import org.apache.spark.streaming.twitter._
import org.apache.spark.storage.StorageLevel
import scala.io.Source
import scala.collection.mutable.HashMap
import java.io.File
import org.apache.log4j.Logger
import org.apache.log4j.Level
import sys.process.stringSeqToProcess
def configureTwitterCredentials(apiKey: String, apiSecret: String, accessToken: String, accessTokenSecret: String) {
  val configs = new HashMap[String, String] ++= Seq(
    "apiKey" -> apiKey, "apiSecret" -> apiSecret, "accessToken" -> accessToken, "accessTokenSecret" -> accessTokenSecret)
  println("Configuring Twitter OAuth")
  configs.foreach{ case(key, value) =>
    if (value.trim.isEmpty) {
      throw new Exception("Error setting authentication - value for " + key + " not set")
    }
    val fullKey = "twitter4j.oauth." + key.replace("api", "consumer")
    System.setProperty(fullKey, value.trim)
    println("\tProperty " + fullKey + " set as [" + value.trim + "]")
  }
  println()
}
val apiKey ="0JCSQxlrWLe3asJL930tlW92Z"
val apiSecret = "ZRD7IqRUHgeQlNkUPHS4GmCcM1zcF3wEb0ZUZB47NYcroLTbkR"
val accessToken = "1061887872997834754-Uacpg4bvOfca4pquPcibbXsKQahFMp"
val accessTokenSecret = "tiaeKQ9YcECE917ZhDKLMdBgPfwqOEkicM6ECavuCtESg"

configureTwitterCredentials(apiKey, apiSecret, accessToken, accessTokenSecret)

val ssc = new StreamingContext(sc, Seconds(15))
val stream = TwitterUtils.createStream(ssc, None)

val hashTags = stream.flatMap(status => status.getText.split(" ").filter(_.startsWith("#")))

val topCounts120 = hashTags.map((_, 1)).reduceByKeyAndWindow(_ + _, Seconds(120)).map{case (topic, count) => (count, topic)}.transform(_.sortByKey(false))
val topCounts30 = hashTags.map((_, 1)).reduceByKeyAndWindow(_ + _, Seconds(30)).map{case (topic, count) => (count, topic)}.transform(_.sortByKey(false))

topCounts120.foreachRDD(rdd => {
  val topList = rdd.take(10)
  println("\nPopular topics in last 120 seconds (%s total):".format(rdd.count()))
  topList.foreach{case (count, tag) => println("%s (%s tweets)".format(tag, count))}
})

topCounts30.foreachRDD(rdd => {
  val topList = rdd.take(10)
  println("\nPopular topics in last 30 seconds (%s total):".format(rdd.count()))
  topList.foreach{case (count, tag) => println("%s (%s tweets)".format(tag, count))}
})
