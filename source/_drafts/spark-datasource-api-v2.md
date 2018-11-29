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

### Limitations of V1 API

## DataSource V2 API

jdbc source, filter push down, partition, write

## References

* http://blog.madhukaraphatak.com/spark-datasource-v2-part-1/
* https://databricks.com/session/apache-spark-data-source-v2
* https://www.slideshare.net/databricks/apache-spark-data-source-v2-with-wenchen-fan-and-gengliang-wang
* https://databricks.com/blog/2015/01/09/spark-sql-data-sources-api-unified-data-access-for-the-spark-platform.html
* https://developer.ibm.com/code/2018/04/16/introducing-apache-spark-data-sources-api-v2/
* https://hackernoon.com/extending-our-spark-sql-query-engine-5f4a088de986
* https://animeshtrivedi.github.io/spark-parquet-reading
* https://michalsenkyr.github.io/2017/02/spark-sql_datasource

[1]: https://github.com/apache/spark/blob/v2.3.2/sql/core/src/main/scala/org/apache/spark/sql/sources/interfaces.scala
