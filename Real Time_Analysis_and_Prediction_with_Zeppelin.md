

<b><i>Real Time Analysis and Prediction with Zeppelin (well..sort of) </b></i>

 Check [Part I](https://github.com/Satanette/Big-Data-Tutorials/blob/master/Spark_Streaming_and_Kafka_integration.md) for more details

<b>Environment</b> based on output from Part I, this tutorial was made entirely in Zeppelin notebooks
http://www.racketracer.com/2015/12/28/getting-started-apache-zeppelin-airbnb-visuals/ 

Let's try to analyze the data directly from the hdfs, since Scala allows to read Spark generated files as csv,
and implement K-means algorithm.

<b>Scala code in Zeppelin notebook</b>

```

%spark
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.commons.io.IOUtils
import java.net.URL
import java.nio.charset.Charset
import org.apache.spark.sql.SQLContext
import org.apache.spark.sql.types._
import org.apache.spark.sql.functions._
import org.apache.spark.sql._
import org.apache.spark.ml.feature.{IndexToString, StringIndexer, VectorIndexer}
import org.apache.spark.ml.feature.VectorIndexer
import org.apache.spark.ml.feature.VectorAssembler
import org.apache.spark.ml.clustering.KMeans

     //create a schema of StructType 
val schema=StructType(Array(
 StructField("state",StringType,true),
 StructField("account_length",DoubleType,true),
 StructField("area_code",StringType,true),
 StructField("phone_number",StringType,true),
 StructField("intl_plan",StringType,true),
 StructField("voice_mail_plan",StringType,true),
 StructField("number_vmail_messages",DoubleType,true),
 StructField("total_day_minutes",DoubleType,true),
 StructField("total_day_calls",DoubleType,true),
 StructField("total_day_charge",DoubleType,true),
 StructField("total_eve_minutes",DoubleType,true),
 StructField("total_eve_calls",DoubleType,true),
 StructField("total_eve_charge",DoubleType,true),
 StructField("total_night_minutes",DoubleType,true),
 StructField("total_night_calls",DoubleType,true),
 StructField("total_night_charge",DoubleType,true),
 StructField("total_intl_minutes",DoubleType,true),
 StructField("total_intl_calls",DoubleType,true),
 StructField("total_intl_charge",DoubleType,true),
 StructField("number_customer_service_calls",DoubleType,true),
 StructField("churned",StringType,true)
))

     //reads Spark generated file as .csv
    val data = spark.read.option("header","false").schema(schema).csv("/tmp/moretesting/file_1495017549227/part-00000")

    data.cache
    data.show
    data.schema    
    
     //where algorithm begins 
     
    val featureCols = Array("total_eve_minutes", "total_night_calls", "total_day_calls")
    val assembler = new VectorAssembler().setInputCols(featureCols).setOutputCol("features")
   
    val data2 = assembler.transform(data)
    
    //train your data - 20% for testing
    val Array(trainingData, testData) = data2.randomSplit(Array(0.8, 0.2)) 
  
   //some kmeans jiu-jitsu; documentation at the ending
   
    val kmeans = new KMeans().setK(8).setFeaturesCol("features").setMaxIter(1)
    val model = kmeans.fit(trainingData)    

    model.clusterCenters.foreach(println)
    val records = model.transform(testData)
   
    //registers dataframe as a temporary table in the SQLContext
    
    records.registerTempTable("dials")

```

 
A snip from data.show 
![ScreenShot](https://github.com/Satanette/test/blob/master/ss5.png)
    
    
Time to analyse and predict the streaming data based on input columns, in sql notebooks for location 
![ScreenShot](https://github.com/Satanette/test/blob/master/ss6.png)


    
    
 <b>Resources and documentation</b>

<i>https://spark.apache.org/docs/2.0.2/ml-clustering.html#k-means</br>
http://spark.apache.org/docs/latest/sql-programming-guide.html</br>
http://blog.cloudera.com/blog/2016/02/how-to-predict-telco-churn-with-apache-spark-mllib/</br>
http://spark.apache.org/docs/latest/sql-programming-guide.html#programmatically-specifying-the-schema</br>
http://www.dummies.com/programming/big-data/data-science/how-to-use-k-means-cluster-algorithms-in-predictive-analysis/</i>
    
    
