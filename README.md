# Apache Spark: What? Why? How? But....
![spark-logo-trademark](https://user-images.githubusercontent.com/1760859/31045086-82ecb952-a5f5-11e7-8208-fd21df1bd409.png)

## Why a new framework (don't we have MapReduce already)?
MapReduce requires data to be serialized to disk between each step, which means that the I/O cost of a MapReduce job is high, making interactive analysis and iterative algorithms very expensive.

## Overview
Spark is a general purpose cluster computing framework that provides efficient in-memory computations for large data sets by distributing computation across multiple computers. 

Any distributed computing framework needs to solve two problems
1. How to distribute data
2. How to distribute computation

Hadoop uses HDFS to solve the distributed data problem and MapReduce as the programming paradigm that provides effective distributed computation. Similarly, Spark has a functional programming API in multiple languages that provides more operators than map and reduce, and does this via a distributed data framework called resilient distributed datasets or RDDs (more on RDDs later). **Spark extends the MapReduce model to support more types of computations and it can cover a wide range of workflows that previously were implemented as specialized systems built on top of Hadoop**. Spark uses in-memory caching to improve performance and, therefore, is fast enough to allow for interactive analysis. Caching also improves the performance of iterative algorithms, which makes it great for data centric tasks, especially machine learning.


## What is RDD anyway?
RDDs are motivated by two types of applications that current computing frameworks handle inefficiently
1. Iterative algorithms
2. Interactive data mining tools

Although current frameworks provide numerous abstractions for accessing a cluster’s computational resources, they lack abstractions for leveraging distributed memory.

RDDs provide an interface based on coarse grained transformations (e.g. map, filter and join) that apply the same operation to many data items. This allows them to efficiently provide fault tolerance by logging the transformations used to build a dataset (its lineage) rather than the actual data. If a partition of an RDD is lost, the RDD has enough information about how it was derived from other RDDs to recompute just that partition. Thus, lost data can be recovered, often quite quickly, without requiring costly replication. This is a powerful property: in essence, **a program cannot reference an RDD that it cannot reconstruct after a failure**.

RDDs are essentially a programming abstraction that represents a read-only collection of objects that are partitioned across machines. RDDs can be rebuilt from a lineage (and are therefore fault tolerant), are accessed via parallel operations, can be read from and written to distributed storages like HDFS or S3, and most importantly, can be cached in the memory of worker nodes for immediate reuse. Because RDDs can be cached in memory, Spark is extremely effective at iterative applications, where the data is being reused throughout the course of an algorithm. Most machine learning and optimization algorithms are iterative, making Spark an extremely effective tool for data science. Additionally, because Spark is so fast, it can be accessed in an interactive fashion via a command line prompt similar to the Python REPL.

RDDs support two types of operations;
1. Transformations, which create a new RDD from an existing one
2. Actions, which return a value to the driver program after running a computation on the RDD.

For example, map() is a transformation that passes each RDD element through a function and returns a new RDD representing the results. On the other hand, reduce() is an action that aggregates all the elements of the RDD using some function and returns the final result to the driver program. Note, however, that there is also a reduceByKey() that returns a distributed dataset.

With aforementioned types of RDD operations, Spark can run more efficiently: a dataset created through map() operation will be used in a consequent reduce() operation and will return only the result of the the last reduce function to the driver. That way, the reduced dataset rather than the larger mapped data set will be returned to the user.

**All transformations in Spark are lazy, in that they do not compute their results right away: instead, they just remember the transformations applied to some base dataset.** The transformations are only computed when an action requires a result to be returned to the driver program.

Here is some code worth explaining;

```
lines = spark.textFile("hdfs://...")
errors = lines.filter(_.startsWith("ERROR"))
// Count errors mentioning MySQL
errors.filter(_.contains("MySQL")).count()
// Return the time fields of errors mentioning
// HDFS as an array (assuming time is field
// number 3 in a tab-separated format):
errors.filter(_.contains("HDFS"))
      .map(_.split(’\t’)(3))
      .collect()
```
After the first action(count) involving errors runs, Spark will store the partitions of errors in memory, greatly speeding up subsequent computations on it. Note that the base RDD, lines, is not loaded into RAM. This is desirable because the error messages might only be a small fraction of the data (small enough to fit into memory).

## (Very Complex) Application Workflow
Programming Spark applications is similar to other data flow languages that had previously been implemented on Hadoop. Code is written in a driver program which is lazily evaluated, and upon an action, the driver code is distributed across the cluster to be executed by workers on their partitions of the RDD. Results are then sent back to the driver for aggregation or compilation. Essentially the driver program creates one or more RDDs, applies operations to transform the RDD, then invokes some action on the transformed RDD. You can develop applications locally and start Spark jobs that will run in a multi-process/multi-threaded mode, or you can configure your machine as a client to a cluster (though this is not recommended as the driver plays an important role in Spark jobs and should be in the same network as the rest of the cluster).

![driver-sparkcontext-clustermanager-workers-executors](https://user-images.githubusercontent.com/1760859/31045087-8305dfa4-a5f5-11e7-8a7e-b0855b2114d7.png)

Image taken from: https://jaceklaskowski.gitbooks.io/mastering-apache-spark

Spark applications are run as independent sets of processes, coordinated by a SparkContext in a driver program. The context will connect to some cluster manager (e.g. YARN) which allocates system resources. Each worker in the cluster is managed by an executor, which is in turn managed by the SparkContext. The executor manages computation as well as storage and caching on each machine. What is important to note is that application code is sent from the driver to the executors, and the executors specify the context and the various tasks to be run. The executors communicate back and forth with the driver for data sharing or for interaction. **Drivers are key participants in Spark jobs, and therefore, they should be on the same network as the cluster.** This is different from Hadoop code, where you might submit a job from anywhere to the JobTracker, which then handles the execution on the cluster.

## Did I Hear SQL?
Spark SQL Provides APIs for interacting with Spark via the Apache Hive variant of SQL called Hive Query Language (HiveQL). Every database table is represented as an RDD and Spark SQL queries are transformed into Spark operations. For those that are familiar with Hive and HiveQL, Spark can act as a drop-in replacement.

## Passing (Secret) Data to Workers

Operations can be invoked on a RDD by passing closure. When Spark runs a closure on a worker, any variables used in the closure are copied to that node, but are maintained within the local scope of that closure.

Spark provides two types of shared variables that can be interacted with by all workers in a restricted fashion.
1. Broadcast variables are distributed to all workers, but are read-only. Broadcast variables can be used as lookup tables or stopword lists.
2. Accumulators are variables that workers can "add" to using associative operations and are typically used as counters.

## (Yet Another) DataFrame

DataFrames are often compared to tables in a relational database or a data frame in R or Python: they have a schema, with column names and types and logic for rows and columns. Because of the disadvantages that you can experience while working with RDDs, the DataFrame API was conceived: it provides you with a higher level abstraction that allows you to use a query language to manipulate the data. This higher level abstraction is a logical plan that represents data and a schema. This means that the frontend to interacting with your data is a lot easier! Because the logical plan will be converted to a physical plan for execution, you’re actually a lot closer to what you’re doing when you’re working with them rather than how you’re trying to do it, because you let Spark figure out the most efficient way to do what you want to do. Remember though that **DataFrames are still built on top of RDDs. And exactly because you let Spark worry about the most efficient way to do things, DataFrames are optimized, and more intelligent decisions will be made when you’re transforming data** and that also explains why they are faster than RDDs.

You can use RDDs when you want to perform low-level transformations and actions on your unstructured data. This means that you don’t care about imposing a schema while processing or accessing the attributes by name or column. In addition, you don’t necessarily need the optimization and performance benefits that DataFrames and DataSets(another high level abstraction on top of RDD) can offer for (semi) structured data.

Note that, even though the Spark, Python and R data frames can be very similar, there are also a lot of differences. Spark DataFrames carry the specific optimization under the hood and can use distributed memory to handle big data, while Pandas DataFrames and R data frames can only run on one computer. However, these differences don’t mean that the two of them can’t work together. You can reuse your existing Pandas DataFrames to scale up to larger data sets. If you want to convert your Spark DataFrame to a Pandas DataFrame and you expect the resulting Pandas’s DataFrame to be small, you can use the following lines of code:
```
df.toPandas() 
```
Note that you do need to make sure that the DataFrame needs to be small enough because all the data is loaded into the driver’s memory.

## Recomputation and Persistence :|

RDDs by default is recomputed each time an action is run on them. For example;
```
lines = sparkContext.textFile("words.txt")
lines.first()
lines.count()
```
Here the call to action first() computes the RDD 'lines'. Again when we use another action 'count()' on the same RDD, the RDD is recomputed once again. The default behavior of recomputing the RDDs on each action can be overridden by persisting the RDDs, so that no re-computation is done each time an action is called on the RDD. When persisted, each node that compute the RDD store the result in their Partitions.

We use persist() method to persist an RDD. In Scala & Java, by default, persist() will store the data in JVM as unserialized object. In Python, calling persist() will serialize the data before persisting. Options to store in Memory/Disk combination is also possible.

```
lines = sc.textFile("words.txt")
lines.persist(StorageLevel.MEMORY_ONLY) //We can also use cache() method if we need MEMORY_ONLY storage level
lines.count()
```

Note that the **actual persistence takes place during the first (count) action call on the RDD.**

Spark provides multiple Storage options(Memory/Disk) to persist the data as well as Replication Levels. More information can be found ![here](https://apachesparkbook.blogspot.com/2015/05/rdd-persistence.html). When the cached data exceeds the Memory capacity, Spark automatically evicts the old partitions(it will be recalculated when needed). This is called Last Recently used Cache(LRU) policy.

A couple of use cases for caching or persisting RDDs are the use of iterative algorithms and fast interactive RDD use. If dataset can be accessed many times or more than one times then RDD should be cached so that recomputation can be faster. If you only read a dataset once there is no point in caching it, it will actually make your job slower. There is very small and purely syntactic difference between caching and persistence of RDDs, the two terms are often used interchangeably. Users can also set a persistence priority on each RDD to specify which in-memory data should spill to disk first.

You can use unpersist() to unpersist a RDD.

## (Not so Useful) Tips
1. Memory leaks can occur if you are careless in persisting, read more here: https://github.com/ContentWise/sparking/wiki/What-if-you-%22.persist()%22-twice

2. By calling collect() on any RDD, you drag data back into your applications from the nodes. Each RDD element will be copy onto the single driver program, which will run out of memory and crash. Given the fact that you want to make use of Spark in the most efficient way possible, it’s not a good idea to call collect() on large RDDs.

3. One of the most basic rules that you can apply when you’re revising the chain of operations that you have written down is to make sure that you filter or reduce your data before joining it. This way, you avoid sending too much data over the network that you’ll throw away after the join. The join operation is one of the most expensive operations that you can use in Spark, so that’s why it makes sense to be wary of this. When you reduce the data before the join, you avoid shuffling your data around too much.

4. On big data sets, you’re better off making use of other functions, such as reduceByKey(), combineByKey() or foldByKey(). When you use groupByKey(), all key-value pairs are shuffled around in the cluster. A lot of unnecessary data is being transferred over the network. Additionally, this also means that if more data is shuffled onto a single machine than can fit in memory, the data will be spilled to disk. This heavily impacts the performance of your Spark job. When you make use of reduceByKey(), for example, the pairs with the same key are already combined before the data is shuffled. As a result, you’ll have to send less data over the network. Next, the reduce function is called again so that all the values from each partition are reduced.

5. Making use of the Spark UI is really something that you can not miss. This web interface allows you to monitor and inspect the execution of your jobs in a web browser, which is extremely important if you want to exercise control over your jobs. The Spark UI allows you to maintain an overview off your active, completed and failed jobs. You can see when you submitted the job, and how long it took for the job to run. Besides the schematic overview, you can also see the event timeline section in the “Jobs” tab. Make sure to also find out more about your jobs by clicking the jobs themselves. You’ll get more information on the stages of tasks inside it. The “Stages” tab in the UI shows you the current stage of all stages of all jobs in a Spark application, while the “Storage” tab will give you more insights on the RDD size and the memory use.

6. RDDs would be less suitable for applications that make asynchronous fine-grained updates to shared state, such as a storage system for a web application or an incremental web crawler. For these applications, it is more efficient to use systems that perform traditional update logging and data check-pointing, such as databases.

## Confession
When I started learning Apache Spark I read material from various websites and took notes (read copy/paste) along the way. Recently, another team at my company started exploring Spark so I decided to share my notes with them (and the world). Here are some of the notes sources
1. ![Paper on RDDs, University of California, Berkeley](https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf)
2. ![Apache Spark Homepage](https://spark.apache.org/)
2. ![Apache Spark in Python: Beginner's Guide, Data Camp](https://www.datacamp.com/community/tutorials/apache-spark-python#gs.iAyDe9M)
3. ![Getting Started with Spark, in Python - District Data Labs](https://districtdatalabs.silvrback.com/getting-started-with-spark-in-python)
4. Stackoverflow
5. Quora
