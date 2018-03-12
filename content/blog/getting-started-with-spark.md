---
title: "Getting Started With Spark"
date: 2018-02-19T14:50:28Z
---

# Running Spark locally (single machine)

## Prerequisites

You must have Java to run Spark.

```console
$ java -version
java version "1.8.0_151"
Java(TM) SE Runtime Environment (build 1.8.0_151-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.151-b12, mixed mode)
```

## Installation

Go to the [Spark Download page](http://spark.apache.org/downloads.html)
and get the latest version, leaving all options to default.
At the time of writing this is `spark-2.2.1-bin-hadoop2.7.tgz`.

```console
$ cd ~
$ mv Downloads/spark-2.2.1-bin-hadoop2.7.tgz .
$ tar -xf spark-2.2.1-bin-hadoop2.7.tgz
$ cd spark-2.2.1-bin-hadoop2.7
```

## Baby steps

Launch the interactive console (Scala by default):

```console
$ ./bin/spark-shell
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
18/02/19 14:56:56 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Spark context Web UI available at http://10.62.0.37:4040
Spark context available as 'sc' (master = local[*], app id = local-1519052217524).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.2.1
      /_/
         
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_151)
Type in expressions to have them evaluated.
Type :help for more information.

scala> 
```

### The Spark Session

The *Spark Application* is controlled through a driver process called [`SparkSession`](https://spark.apache.org/docs/2.2.1/api/java/index.html?org/apache/spark/sql/SparkSession.html).

The `SparkSession` instance ensures appropriate execution across the cluster. 
Even with multiple nodes in the cluster, one `SparkSession` corresponds to one *Spark Application*. 

This instance is available as the `spark` variable in Python and Scala when using the console.

```console
scala> spark
res0: org.apache.spark.sql.SparkSession = org.apache.spark.sql.SparkSession@243c4346

scala>
```

### DataFrames

[`DataFrames`](https://spark.apache.org/docs/latest/sql-programming-guide.html#datasets-and-dataframes) 
are the most common type of data structure in Spark.

They are essentially like relational tables: a collection of named columns.

Let's create one!
We will create here a single-column `DataFrame`, called `number`, with numbers from `0` to `999`.

```console
scala> val myRange = spark.range(1000).toDF("number")
18/02/19 15:23:26 WARN ObjectStore: Failed to get database global_temp, returning NoSuchObjectException
myRange: org.apache.spark.sql.DataFrame = [number: bigint]
```

Let's show this now:
```console
scala> myRange.show()
+------+
|number|
+------+
|     0|
|     1|
|     2|
|     3|
|     4|
|     5|
|     6|
|     7|
|     8|
|     9|
|    10|
|    11|
|    12|
|    13|
|    14|
|    15|
|    16|
|    17|
|    18|
|    19|
+------+
only showing top 20 rows
```

We can also show its schema (which in this case has been auto-guessed from supplied data):

```console
scala> myRange.printSchema()
root
 |-- number: long (nullable = false)
```

This range of numbers is called a *distributed collection*, and as its name suggests, when Spark runs on a cluster, 
executors will have a sub-set of it.

This allows tasks to run in parallel. 
Each chunk is called *partition*.

Don't be afraid of all that as you won't even have to care about it for the most part, Spark will do it for you.

### Transformations

> In Spark, core data structure are [*immutable*](https://en.wikipedia.org/wiki/Immutable_object), 
so they cannot be changed after they have been created. 

Transformations are *lazy*, which means that they are defined but not executed immediately.
They are executed only when an *action* is requested.

Consider the following example, filtering out our list of numbers which are even (there is no remainder when dividing it by 2). 

```console
scala> val divisBy2 = myRange.where("number % 2 = 0")
divisBy2: org.apache.spark.sql.Dataset[org.apache.spark.sql.Row] = [number: bigint]

scala> 
``` 

The actual computation of it would happen only when we:

```console
scala> divisBy2.show()
+------+
|number|
+------+
|     0|
|     2|
|     4|
|     6|
|     8|
|    10|
|    12|
|    14|
|    16|
|    18|
|    20|
|    22|
|    24|
|    26|
|    28|
|    30|
|    32|
|    34|
|    36|
|    38|
+------+
only showing top 20 rows
```

Or
```console
scala> divisBy2.count()
res7: Long = 500
```

The benefit of not executing transformations immediately is:

* we can chain transformations (*pipelines*)
* we're effectively building a *plan*, which allows Spark to start organising itself for optimal execution

There are 2 types of transformations:

* *narrow transformations*: One input partition gives one and only one output partition (this is the case for our `where` transformation)
* *wide transformation*: The data required to compute the records in a single partition may reside in many partitions of the parent.

### More data!

Open a new terminal and download a CSV example file to `/tmp`:

```console
$ wget -P /tmp https://raw.githubusercontent.com/databricks/Spark-The-Definitive-Guide/master/data/flight-data/csv/2015-summary.csv
```

Verify what's in there:

```console
$ head -n 5 /tmp/2015-summary.csv 
DEST_COUNTRY_NAME,ORIGIN_COUNTRY_NAME,count
United States,Romania,15
United States,Croatia,1
United States,Ireland,344
Egypt,United States,15
```

> Here, `head` returns the first `n` lines of the file specified (the CVS headers and then 4 lines of data)

Back to Scala:

Instruct our Scala Application about this file, specifying:

* `inferSchema` set to `true`: Scala will do its best guess to deduce the data's schema
* `header` set to `true`: means that the first line of the csv is the header, so it can be used as columns names

```console
scala> val flightData2015 = spark
         .read
         .option("inferSchema", "true")
         .option("header", "true")
         .csv("/tmp/2015-summary.csv")
```

Note that the `read` method is a transformation and as such is *lazy*, so the `DataFrames` here have an unknown length.
Scala just read a few lines at the top to infer schema, but that's it.

We need to apply an action to get some results, such as returning the top 4 records:

```console
scala> flightData2015.take(4)
res1: Array[org.apache.spark.sql.Row] = Array([United States,Romania,15], [United States,Croatia,1], [United States,Ireland,344], [Egypt,United States,15])
```

> These records effectively correspond to the ones we saw with the `head` command.

Let's sort these results by `count` (last column) and explain what Spark will do (what's the *plan*):

```console
scala> flightData2015.sort("count").explain()
== Physical Plan ==
*Sort [count#14 ASC NULLS FIRST], true, 0
+- Exchange rangepartitioning(count#14 ASC NULLS FIRST, 200)
   +- *FileScan csv [DEST_COUNTRY_NAME#12,ORIGIN_COUNTRY_NAME#13,count#14] Batched: false, Format: CSV, Location: InMemoryFileIndex[file:/tmp/2015-summary.csv], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<DEST_COUNTRY_NAME:string,ORIGIN_COUNTRY_NAME:string,count:int>
```

This has to be read from bottom to top, top being the last executed operation and bottom the first one.

Spark then:

* `FileScan csv`: Scans the file
* `Exchange rangepartitioning` has to be executed prior to `sort` as it's a wide transformation, so data partitions have to be reorganised
* `Sort`, finally

We can confirm by retrieving the first 4 as we did before:

```console
scala> flightData2015.sort("count").take(4)
res7: Array[org.apache.spark.sql.Row] = Array([Moldova,United States,1], [United States,Singapore,1], [United States,Croatia,1], [Malta,United States,1])
```

### A more advanced example:

```console
scala> :paste
// Entering paste mode (ctrl-D to finish)

flightData2015
  .groupBy("DEST_COUNTRY_NAME")
  .sum("count")
  .withColumnRenamed("sum(count)", "destination_total")
  .sort(desc("destination_total"))
  .limit(5)
  .show()

// Exiting paste mode, now interpreting.

+-----------------+-----------------+                                           
|DEST_COUNTRY_NAME|destination_total|
+-----------------+-----------------+
|    United States|           411352|
|           Canada|             8399|
|           Mexico|             7140|
|   United Kingdom|             2025|
|            Japan|             1548|
+-----------------+-----------------+


scala> 
```

> Notice the `:paste` keyword, allowing us to paste multiple-line statements. 
To use it, just type `:paste` and press enter, then follow instructions (ctrl+d to exit)
