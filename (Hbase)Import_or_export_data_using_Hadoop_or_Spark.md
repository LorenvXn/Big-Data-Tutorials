<i>This tutorial will focus strictly on Java code and command lines.</br>
The Hbase Browser will be used only for checking or confirming results</i>

<i><b>Creating table in Hbase</i></b>

<b>Option 1: Hbase shell</b>

![ScreenShot](https://github.com/Satanette/test/blob/master/sp1.png)

Simple example how to create a table with 2 rows, from Hbase shell.
Please be aware that data in Hbase is not increasing vertically, but horizontally.

![ScreenShot](https://github.com/Satanette/test/blob/master/sp2.png)

Check Hbase Browser to confirm:

![ScreenShot](https://github.com/Satanette/test/blob/master/sp3.png)

<b>Option 2: Java Code</b>

Below code, will create table <i>bookz</i>:

```
import java.io.IOException;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.HColumnDescriptor;
import org.apache.hadoop.hbase.HTableDescriptor;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Admin;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.conf.Configuration;

public class createBookz {

 public static void main(String[] args) throws IOException {
 
    Configuration conf = HBaseConfiguration.create();
    Connection conn=ConnectionFactory.createConnection(conf);
    
    Admin admin=conn.getAdmin();
    
    HTableDescriptor tableDescriptor = new
    HTableDescriptor(TableName.valueOf("bookz"));
    tableDescriptor.addFamily(new HColumnDescriptor("author"));
    tableDescriptor.addFamily(new HColumnDescriptor("title"));
    admin.createTable(tableDescriptor);
    System.out.println("ditto with yo table!");
 }
}
```

And now, let's insert some data in bookz:

```
import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.client.Table;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.util.Bytes;

public class new_stuff_to_insert {
      public static void main(String[] args) throws IOException {
      Configuration conf = HBaseConfiguration.create();
      Connection connection = ConnectionFactory.createConnection(conf);
      Table hive_table = connection.getTable(TableName.valueOf("bookz"));
      Put m  = new Put(Bytes.toBytes("1"));
      m.add(Bytes.toBytes("author"), (Bytes.toBytes("name")), (Bytes.toBytes("Stanislaw Lem")));
      m.add(Bytes.toBytes("title"), (Bytes.toBytes("title")), (Bytes.toBytes("Solaris")));
      hive_table.put(m);
      System.out.println("INSERTED!!!");
      hive_table.close();
 }
}
``` 
Compile with:
```
javac -cp `hbase classpath` bookz.java
java -cp `hbase classpath` bookz
Let's check the Hbase Browser:
```
![ScreenShot](https://github.com/Satanette/test/blob/master/sp4.png)

<i>More details about Java packages and Hbase at this link:
https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/package-summary.html</i>


<b><i>Hadoop and Hbase</i></b>

<i>These examples are based on package org.apache.hadoop.hbase.mapreduce</i>

<b>First example</b>: Counting rows from a table using mapreduce packages

![ScreenShot](https://github.com/Satanette/test/blob/master/sp5.png)

Let's check the Jobhistory Server, at port 19888, and see if the job job_1491557321413_0003 has been implemented successfully:

![ScreenShot](https://github.com/Satanette/test/blob/master/sp6.png)

<b>Second Example</b>: Export/Import table with Hadoop

This is a very easy one (example for export):
```
sudo -u hdfs hbase org.apache.hadoop.hbase.mapreduce.Export bookz /user/export
```
Make sure the folder is not created already. If everything OK, all should be looking as below

![ScreenShot](https://github.com/Satanette/test/blob/master/sp7.png)

Documentation: https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/mapreduce/package-summary.html


<i><b>Spark and Hbase</i></b>

We will have to write another small Java code for creating a table. This time only, we will be creating a jar,
and create the table by using spark-submit:

Let's create “albumz” table, using createAlbums.java . 
The steps for creating this using Java are almost as those used for bookz table, then again, I`ll put the code here for Spark output comparison:

```
import java.io.IOException;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.HColumnDescriptor;
import org.apache.hadoop.hbase.HTableDescriptor;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Admin;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.conf.Configuration;
public class createAlbums {
 public static void main(String[] args) throws IOException {
    Configuration conf = HBaseConfiguration.create();
    Connection conn=ConnectionFactory.createConnection(conf);
    Admin admin=conn.getAdmin();
    HTableDescriptor tableDescriptor = new
    HTableDescriptor(TableName.valueOf("albumz"));
    tableDescriptor.addFamily(new HColumnDescriptor("album"));
    tableDescriptor.addFamily(new HColumnDescriptor("title"));
    admin.createTable(tableDescriptor);
    System.out.println("ditto with yo album!");
 }
}

```

<b>1) Build the jar</b>

a) Create a new directory, under the root project where the java code is. In this example, new_build

b) Create the class under the folder:
```
javac -cp `hbase classpath` albumz.java -d /root/test/new_build
```
c) … and create the jar:
```
jar -cvf HadoopJava.jar .
```
It should be looking as below:

![ScreenShot](https://github.com/Satanette/test/blob/master/sp8.png)

<b>2)Submit it!</b>
![ScreenShot](https://github.com/Satanette/test/blob/master/sp9.png)

Table successfully built using Spark.

<b>3) Check Hbase Browser to see if it is created</b>
![ScreenShot](https://github.com/Satanette/test/blob/master/sp10.png)


