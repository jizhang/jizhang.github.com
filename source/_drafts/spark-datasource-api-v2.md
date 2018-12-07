---
title: Spark DataSource API V2
tags: [spark]
categories: Big Data
---

From Spark 1.3, the team introduced a data source API to help quickly integrating various input formats with Spark SQL. But eventually this version of API became insufficient and the team needed to add a lot of internal codes to provide more efficient solutions for Spark SQL data sources. So in Spark 2.3, the second version of data source API is out, which is supposed to overcome the limitations of the previous version. In this article, I will demonstrate how to implement custom data source for Spark SQL in both V1 and V2 API, to help understanding their differences and the new API's advantages.

## DataSource V1 API

V1 API provides a set of abstract classes and traits. They are located in [spark/sql/sources/interfaces.scala][1]. Some basic APIs are:

```scala
trait RelationProvider {
  def createRelation(sqlContext: SQLContext, parameters: Map[String, String]): BaseRelation
}

abstract class BaseRelation {
  def sqlContext: SQLContext
  def schema: StructType
}

trait TableScan {
  def buildScan(): RDD[Row]
}
```

A `RelationProvider` defines a class that can create a relational data source for Spark SQL to manipulate with. It can initialize itself with provided options, such as file path or authentication. `BaseRelation` is used to define the data schema, which can be loaded from database, Parquet file, or specified by the user. This class also needs to mix-in one of the `Scan` traits, implements the `buildScan` method, and returns an RDD.

<!-- more -->

### JdbcSourceV1

Now we use V1 API to implement a JDBC data source. For simplicity, the table schema is hard coded, and it only supports full table scan. Complete example can be found on GitHub ([link][2]), while the sample data is in [here][3].

```scala
class JdbcSourceV1 extends RelationProvider {
  override def createRelation(parameters: Map[String, String]): BaseRelation = {
    new JdbcRelationV1(parameters("url"))
  }
}

class JdbcRelationV1(url: String) extends BaseRelation with TableScan {
  override def schema: StructType = StructType(Seq(
    StructField("id", IntegerType),
    StructField("emp_name", StringType)
  ))

  override def buildScan(): RDD[Row] = new JdbcRDD(url)
}

class JdbcRDD(url: String) extends RDD[Row] {
  override def compute(): Iterator[Row] = {
    val conn = DriverManager.getConnection(url)
    val stmt = conn.prepareStatement("SELECT * FROM employee")
    val rs = stmt.executeQuery()

    new Iterator[Row] {
      def hasNext: Boolean = rs.next()
      def next: Row = Row(rs.getInt("id"), rs.getString("emp_name"))
    }
  }
}
```

The actual data reading happens in `JdbcRDD#compute`. It receives the connection options, possibly with pruned column list and where conditions, executes the query, and returns an iterator of `Row` objects, correspondent to the defined schema. Now we can create a `DataFrame` from this custom data source.

```scala
val df = spark.read
  .format("JdbcSourceV2")
  .option("url", "jdbc:mysql://localhost/spark")
  .load()

df.printSchema()
df.show()
```

The outputs are:

```text
root
 |-- id: integer (nullable = true)
 |-- emp_name: string (nullable = true)
 |-- dep_name: string (nullable = true)
 |-- salary: decimal(7,2) (nullable = true)
 |-- age: decimal(3,0) (nullable = true)

+---+--------+----------+-------+---+
| id|emp_name|  dep_name| salary|age|
+---+--------+----------+-------+---+
|  1| Matthew|Management|4500.00| 55|
|  2|  Olivia|Management|4400.00| 61|
|  3|   Grace|Management|4000.00| 42|
|  4|     Jim|Production|3700.00| 35|
|  5|   Alice|Production|3500.00| 24|
+---+--------+----------+-------+---+
```

### Limitations of V1 API

As we can see, V1 API is quite straightforward and can meet the initial requirements of Spark SQL use cases. But as Spark moves forward, V1 API starts to show its limitations.

#### Coupled with Higher Level API

`createRelation` accepts `SQLContext` as parameter; `buildScan` returns `RDD` of `Row`; and when implementing writable data source, the `insert` method accepts `DataFrame` type.

```scala
trait InsertableRelation {
  def insert(data: DataFrame, overwrite: Boolean): Unit
}
```

These classes are of higher level of Spark API, and some of them have already upgraded, like `SQLContext` is superceded by `SparkSession`, and `DataFrame` is now an alias of `Dataset[Row]`. Data sources should not be required to reflect these changes.

#### Hard to Add New Push Down Operators

Besides `TableScan`, V1 API provides `PrunedScan` to eliminate unnecessary columns, and `PrunedFilteredScan` to push predicates down to data source. In `JdbcSourceV1`, they are reflected in the SQL statement.

```scala
class JdbcRelationV1 extends BaseRelation with PrunedFilteredScan {
  def buildScan(requiredColumns: Array[String], filters: Array[Filter]) = {
    new JdbcRDD(requiredColumns, filters)
  }
}

class JdbcRDD(columns: Array[String], filters: Array[Filter]) {
  def compute() = {
    val wheres = filters.flatMap {
      case EqualTo(attribute, value) => Some(s"$attribute = '$value'")
      case _ => None
    }
    val sql = s"SELECT ${columns.mkString(", ")} FROM employee WHERE ${wheres.mkString(" AND ")}"
  }
}
```

What if we need to push down a new operator like `limit`? It will introduce a whole new set of `Scan` traits.

```scala
trait LimitedScan {
  def buildScan(limit: Int): RDD[Row]
}

trait PrunedLimitedScan {
  def buildScan(requiredColumns: Array[String], limit: Int): RDD[Row]
}

trait PrunedFilteredLimitedScan {
  def buildScan(requiredColumns: Array[String], filters: Array[Filter], limit: Int): RDD[Row]
}
```

#### Hard to Pass Partition Info

For data sources that support partitioning like HDFS and Kafka, V1 API does not provide native support for partitioning and data locality. We need to achieve this by extending the RDD class. For instance, some Kafka topic contains several partitions, and we want the data reading task to be run on the servers where leader brokers reside.

```scala
case class KafkaPartition(partitionId: Int, leaderHost: String) extends Partition {
  def index: Int = partitionId
}

class KafkaRDD(sc: SparkContext) extends RDD[Row](sc, Nil) {
  def getPartitions: Array[Partition] = Array(
    // populate with Kafka PartitionInfo
    KafkaPartition(0, "broker_0"),
    KafkaPartition(1, "broker_1")
  )

  override def getPreferredLocations(split: Partition): Seq[String] = Seq(
    split.asInstanceOf[KafkaPartition].leaderHost
  )
}
```

Besides, some database like Cassandra distributes data by primary key. If the query pipeline contains grouping on the columns, this information can be used by the optimizer to avoid shuffling. V2 API supports this with a dedicated trait.

#### Lack of Transactional Writing

Spark tasks may fail, and with V1 API there will be partially written data. For file systems like HDFS, we can put a `_SUCCESS` file in the output directory to indicate if the job finishes successfully, but this process needs to be implemented by users, while V2 API provides explicit interfaces to support transactional writing.

#### Lack of Columnar and Streaming Support

Columnar data and stream processing are both added to Spark SQL without using V1 API. Current implementations like `ParquetFileFormat` and `KafkaSource` are written in dedicated codes with internal APIs. These features are also addressed by V2 API.

## DataSource V2 API

V2 API starts with a marker interface `DataSourceV2`. The class needs to be mixed-in with either `ReadSupport` or `WriteSupport`. `ReadSupport` interface, for instance, creates a `DataSourceReader` with initialization options; `DataSourceReader` reads schema of the data source, and returns a list of `DataReaderFactory`; the factory will create the actual `DataReader`, which works like an iterator. Besides, `DataSourceReader` can mix-in various `Support` interfaces, to apply query optimizations like operator push-down and columnar scan. For `WriteSupport` interfaces, the hierarchy is similar. All of them are written in Java for better interoperability.

```java
public interface DataSourceV2 {}

public interface ReadSupport extends DataSourceV2 {
  DataSourceReader createReader(DataSourceOptions options);
}

public interface DataSourceReader {
  StructType readSchema();
  List<DataReaderFactory<Row>> createDataReaderFactories();
}

public interface SupportsPushDownRequiredColumns extends DataSourceReader {
  void pruneColumns(StructType requiredSchema);
}

public interface DataReaderFactory<T> {
  DataReader<T> createDataReader();
}

public interface DataReader<T> extends Closeable {
  boolean next();
  T get();
}
```

You may notice that `DataSourceReader#createDataReaderFactories` still relies on `Row` class, because currently only `Row` is supported, and V2 API is still marked as `Evolving`.

### JdbcSourceV2

Let us rewrite the JDBC data source with V2 API. The following an abridged example of full table scan. Complete code can be found on GitHub ([link][4]).

```scala
class JdbcDataSourceReader extends DataSourceReader {
  def readSchema = StructType(Seq(
    StructField("id", IntegerType),
    StructField("emp_name", StringType)
  ))

  def createDataReaderFactories() = {
    Seq(new JdbcDataReaderFactory(url)).asJava
  }
}

class JdbcDataReader(url: String) extends DataReader[Row] {
  private var conn: Connection = null
  private var rs: ResultSet = null

  def next() = {
    if (rs == null) {
      conn = DriverManager.getConnection(url)
      val stmt = conn.prepareStatement("SELECT * FROM employee")
      rs = stmt.executeQuery()
    }
    rs.next()
  }

  def get() = Row(rs.getInt("id"), rs.getString("emp_name"))
}
```

#### Prune Columns

`DataSourceReader` can mix-in the `SupportsPushDownRequiredColumns` trait. Spark will invoke the `pruneColumns` method with required `StructType`, and `DataSourceReader` can pass it to underlying `DataReader`.

```scala
class JdbcDataSourceReader with SupportsPushDownRequiredColumns {
  var requiredSchema = JdbcSourceV2.schema
  def pruneColumns(requiredSchema: StructType)  = {
    this.requiredSchema = requiredSchema
  }

  def createDataReaderFactories() = {
    val columns = requiredSchema.fields.map(_.name)
    Seq(new JdbcDataReaderFactory(columns)).asJava
  }
}
```

We can examine the execution plan with `df.explain(true)`. For instance, the optimized logical plan of query `SELECT emp_name, age FROM employee` shows column pruning is pushed down to the data source.

```text
== Analyzed Logical Plan ==
emp_name: string, age: decimal(3,0)
Project [emp_name#1, age#4]
+- SubqueryAlias employee
   +- DataSourceV2Relation [id#0, emp_name#1, dep_name#2, salary#3, age#4], datasource.JdbcDataSourceReader@15ceeb42

== Optimized Logical Plan ==
Project [emp_name#1, age#4]
+- DataSourceV2Relation [emp_name#1, age#4], datasource.JdbcDataSourceReader@15ceeb42
```

#### Push Down Filters

Similarly, with `SupportsPushDownFilters`, we can add where conditions to the underlying SQL query.

```scala
class JdbcDataSourceReader with SupportsPushDownFilters {
  var filters = Array.empty[Filter]
  var wheres = Array.empty[String]

  def pushFilters(filters: Array[Filter]) = {
    val supported = ListBuffer.empty[Filter]
    val unsupported = ListBuffer.empty[Filter]
    val wheres = ListBuffer.empty[String]

    filters.foreach {
      case filter: EqualTo => {
        supported += filter
        wheres += s"${filter.attribute} = '${filter.value}'"
      }
      case filter => unsupported += filter
    }

    this.filters = supported.toArray
    this.wheres = wheres.toArray
    unsupported.toArray
  }

  def pushedFilters = filters

  def createDataReaderFactories() = {
    Seq(new JdbcDataReaderFactory(wheres)).asJava
  }
}
```

#### Multiple Partitions

`createDataReaderFactories` returns a list. Each reader will output data for an RDD partition. Say we want to parallelize the data reading tasks, we can divide the records into to two parts, according to primary key ranges.

```scala
def createDataReaderFactories() = {
  Seq((1, 6), (7, 11)).map { case (minId, maxId) =>
    val partition = s"id BETWEEN $minId AND $maxId"
    new JdbcDataReaderFactory(partition)
  }.asJava
}
```

### Transactional Write

hdfs file, diagram, unit test codes?

### Columnar

parquet

### Streaming

Kafka

## References

* http://blog.madhukaraphatak.com/spark-datasource-v2-part-1/
* https://databricks.com/session/apache-spark-data-source-v2
* https://databricks.com/blog/2015/01/09/spark-sql-data-sources-api-unified-data-access-for-the-spark-platform.html
* https://developer.ibm.com/code/2018/04/16/introducing-apache-spark-data-sources-api-v2/
* https://hackernoon.com/extending-our-spark-sql-query-engine-5f4a088de986
* https://animeshtrivedi.github.io/spark-parquet-reading
* https://michalsenkyr.github.io/2017/02/spark-sql_datasource

https://www.slideshare.net/databricks/apache-spark-data-source-v2-with-wenchen-fan-and-gengliang-wang

[1]: https://github.com/apache/spark/blob/v2.3.2/sql/core/src/main/scala/org/apache/spark/sql/sources/interfaces.scala
[2]: https://github.com/jizhang/spark-sandbox/blob/master/src/main/scala/datasource/JdbcExampleV1.scala
[3]: https://github.com/jizhang/spark-sandbox/blob/master/data/employee.sql
[4]: https://github.com/jizhang/spark-sandbox/blob/master/src/main/scala/datasource/JdbcExampleV2.scala
