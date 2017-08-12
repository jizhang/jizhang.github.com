---
title: Extract Data from MySQL with Binlog and Canal
tags:
  - etl
  - mysql
  - canal
  - java
categories:
  - Big Data
date: 2017-08-12 19:15:09
---


Data extraction is the very first step of an ETL process. We need to load data from external data stores like RDMBS or logging file system, and then we can do cleaning, transformation and summary. In modern website stack, MySQL is the most widely used database, and it's common to extract data from different instances and load into a central MySQL database, or directly into Hive. There're several query-based techniques that we can use to do the extraction, including the popular open source software [Sqoop][1], but they are not meant for real-time data ingestion. Binlog, on the other hand, is a real-time data stream that is used to do replication between master and slave instances. With the help of Alibaba's open sourced [Canal][2] project, we can easily utilize the binlog facility to do data extraction from MySQL database to various destinations.

![Canal](/images/canal.png)

## Canal Components

In brief, Canal simulates itself to be a MySQL slave and dump binlog from master, parse it, and send to downstream sinks. Canal consists of two major components, namely Canal server and Canal client. A Canal server can connect to multiple MySQL instances, and maintains an event queue for each instance. Canal clients can then subscribe to theses queues and receive data changes. The following is a quick start guide to get Canal going.

<!-- more -->

### Configure MySQL Master

MySQL binlog is not enabled by default. Locate your `my.cnf` file and make these changes:

```properties
server-id = 1
log_bin = /path/to/mysql-bin.log
binlog_format = ROW
```

Note that `binlog_format` must be `ROW`, becuase in `STATEMENT` or `MIXED` mode, only SQL statements will be logged and transferred (to save log size), but what we need is full data of the changed rows.

Slave connects to master via an dedicated account, which must have the global `REPLICATION` priviledges. We can use the `GRANT` statement to create the account:

```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT
ON *.* TO 'canal'@'%' IDENTIFIED BY 'canal';
```

### Startup Canal Server

Download Canal server from its GitHub Releases page ([link][3]). The config files reside in `conf` directory. A typical layout is:

```text
canal.deployer/conf/canal.properties
canal.deployer/conf/instanceA/instance.properties
canal.deployer/conf/instanceB/instance.properties
```

In `conf/canal.properties` there's the main configuration. `canal.port` for example defines which port Canal server is listening. `instanceA/instance.properties` defines the MySQL instance that Canal server will draw binlog from. Important settings are:

```properties
# slaveId cannot collide with the server-id in my.cnf
canal.instance.mysql.slaveId = 1234
canal.instance.master.address = 127.0.0.1:3306
canal.instance.dbUsername = canal
canal.instance.dbPassword = canal
canal.instance.connectionCharset = UTF-8
# process all tables from all databases
canal.instance.filter.regex = .*\\..*
```

Start the server by `sh bin/startup.sh`, and you'll see the following output in `logs/example/example.log`:

```text
Loading properties file from class path resource [canal.properties]
Loading properties file from class path resource [example/instance.properties]
start CannalInstance for 1-example
[destination = example , address = /127.0.0.1:3306 , EventParser] prepare to find start position just show master status
```

### Write Canal Client

To consume update events from Canal server, we can create a Canal client in our application, specify the instance and tables we're interested in, and start polling.

First, add `com.alibaba.otter:canal.client` dependency to your `pom.xml`, and construct a Canal client:

```java
CanalConnector connector = CanalConnectors.newSingleConnector(
        new InetSocketAddress("127.0.0.1", 11111), "example", "", "");

connector.connect();
connector.subscribe(".*\\..*");

while (true) {
    Message message = connector.getWithoutAck(100);
    long batchId = message.getId();
    if (batchId == -1 || message.getEntries().isEmpty()) {
        Thread.sleep(3000);
    } else {
        printEntries(message.getEntries());
        connector.ack(batchId);
    }
}
```

The code is quite similar to consuming from a message queue. The update events are sent in batches, and you can acknowledge every batch after being properly processed.

```java
// printEntries
RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
for (RowData rowData : rowChange.getRowDatasList()) {
    if (rowChange.getEventType() == EventType.INSERT) {
      printColumns(rowData.getAfterCollumnList());
    }
}
```

Every `Entry` in a message represents a set of row changes with the same event type, e.g. INSERT, UPDATE, or DELETE. For each row, we can get the column data like this:

```java
// printColumns
String line = columns.stream()
        .map(column -> column.getName() + "=" + column.getValue())
        .collect(Collectors.joining(","));
System.out.println(line);
```

Full example can be found on GitHub ([link][4]).

## Load into Data Warehouse

### RDBMS with Batch Insert

For DB based data warehouse, we can directly use the `REPLACE` statement and let the database deduplicates rows by primary key. One concern is the instertion performance, so it's often necessary to cache the data for a while and do a batch insertion, like:

```sql
REPLACE INTO `user` (`id`, `name`, `age`, `updated`) VALUES
(1, 'Jerry', 30, '2017-08-12 16:00:00'),
(2, 'Mary', 28, '2017-08-12 17:00:00'),
(3, 'Tom', 36, '2017-08-12 18:00:00');
```

Another approach is to write the extracted data into a delimited text file, then execute a `LOAD DATA` statement. These files can also be used to import data into Hive. But for both approaches, make sure you escape the string columns properly, so as to avoid insertion errors.

### Hive-based Warehouse

Hive tables are stored on HDFS, which is an append-only file system, so it takes efforts to update data in a previously loaded table. One can use a JOIN-based approach, Hive transaction, or switch to HBase.

Data can be categorized into base and delta. For example, yesterday's `user` table is the base, while today's updated rows are the delta. Using a `FULL OUTER JOIN` we can generate the latest snapshot:

```sql
SELECT
  COALESCE(b.`id`, a.`id`) AS `id`
  ,COALESCE(b.`name`, a.`name`) AS `name`
  ,COALESCE(b.`age`, a.`age`) AS `age`
  ,COALESCE(b.`updated`, a.`updated`) AS `updated`
FROM dw_stage.`user` a
FULL OUTER JOIN (
  -- deduplicate by selecting the latest record
  SELECT `id`, `name`, `age`, `updated`
  FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY `id` ORDER BY `updated` DESC) AS `n`
    FROM dw_stage.`user_delta`
  ) b
  WHERE `n` = 1
) b
ON a.`id` = b.`id`;
```

Hive 0.13 introduces transaction and ACID table, 0.14 brings us the `INSERT`, `UPDATE` and `DELETE` statements, and Hive 2.0.0 provides a new [Streaming Mutation API][5] that can be used to submit insert/update/delete transactions to Hive tables programmatically. Currently, ACID tables must use ORC file format, and be bucketed by primiary key. Hive will store the mutative operations in delta files. When reading from this table, `OrcInputFormat` will figure out which record is the latest. The official sample code can be found in the test suite ([link][7]).

And the final approach is to use HBase, which is a key-value store built on HDFS, making it perfect for data updates. Its table can also be used by MapReduce jobs, or you can create an external Hive table that points directly to HBase. More information can be found on the [official website][6].

## Initialize Target Table

Data extraction is usually on-demand, so there may be already historical data in the source table. One obvious approach is dumping the full table manually and load into destination. Or we can reuse the Canal facility, notify the client to query data from source and do the updates.

First, we create a helper table in the source database:

```sql
CREATE TABLE `retl_buffer` (
  id BIGINT AUTO_INCREMENT PRIMARY KEY
  ,table_name VARCHAR(255)
  ,pk_value VARCHAR(255)
);
```

To reload all records in `user` table:

```sql
INSERT INTO `retl_buffer` (`table_name`, `pk_value`)
SELECT 'user', `id` FROM `user`;
```

When Canal client receives the `RowChange` of `retl_buffer` table, it can extract the table name and primary key value from the record, query the source database, and write to the destination.

```java
if ("retl_buffer".equals(entry.getHeader().getTableName())) {
    String tableName = rowData.getAfterColumns(1).getValue();
    String pkValue = rowData.getAfterColumns(2).getValue();
    System.out.println("SELECT * FROM " + tableName + " WHERE id = " + pkValue);
}
```

This approach is included in another Alibaba's project [Otter][8].

## Canal High Availability

* Canal instances can be supplied with a standby MySQL source, typically in a Master-Master HA scenario. Make sure you turn on the `log_slave_updates` option in both MySQL instances. Canal uses a dedicated heartbeat check, i.e. update a row periodically to check if current source is alive.
* Canal server itself also supports HA. You'll need a Zookeeper quorum to enable this feature. Clients will get the current server location from Zookeeper, and the server will record the last binlog offset that has been consumed.

For more information, please checkout the [AdminGuide][9].

## References

* https://github.com/alibaba/canal/wiki (in Chinese)
* https://github.com/alibaba/otter/wiki (in Chinese)
* https://www.phdata.io/4-strategies-for-updating-hive-tables/
* https://hortonworks.com/blog/four-step-strategy-incremental-updates-hive/
* https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions

[1]: http://sqoop.apache.org/
[2]: https://github.com/alibaba/canal
[3]: https://github.com/alibaba/canal/releases
[4]: https://github.com/jizhang/java-sandbox/blob/blog-canal/src/main/java/com/shzhangji/javasandbox/canal/SimpleClient.java
[5]: https://cwiki.apache.org/confluence/display/Hive/HCatalog+Streaming+Mutation+API
[6]: http://hbase.apache.org/
[7]: https://github.com/apache/hive/blob/master/hcatalog/streaming/src/test/org/apache/hive/hcatalog/streaming/mutate/ExampleUseCase.java
[8]: https://github.com/alibaba/otter/wiki/Manager%E9%85%8D%E7%BD%AE%E4%BB%8B%E7%BB%8D#%E8%87%AA%E5%AE%9A%E4%B9%89%E6%95%B0%E6%8D%AE%E5%90%8C%E6%AD%A5%E8%87%AA-%E7%94%B1-%E9%97%A8
[9]: https://github.com/alibaba/canal/wiki/AdminGuide
