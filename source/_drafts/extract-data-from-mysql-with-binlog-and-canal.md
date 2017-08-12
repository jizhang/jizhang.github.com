---
title: Extract Data from MySQL with Binlog and Canal
tags: [etl, mysql, canal, java]
categories: [Big Data]
---

Data extraction is the very first step of an ETL process. We need to load data from external data stores like RDMBS or logging file system, and then we can do cleaning, transformation and summary. In modern website stack, MySQL is the most widely used database, and it's common to extract data from different instances and load into a central MySQL database, or directly into Hive. There're several query-based techniques that we can use to do the extraction, including the popular open source software [Sqoop][1], but they are not meant for real-time data ingestion. Binlog, on the other hand, is a real-time data stream that is used to do replication between master and slave instances. With the help of Alibaba's open sourced [Canal][2] project, we can easily utilize the binlog facility to do data extraction from MySQL database to various destinations.

![Canal](/images/canal.png)

## Canal Components

In brief, Canal simulates itself to be a MySQL slave and dump binlog from master, parse it, and send to downstream sinks. Canal consists of two major components, i.e. Canal server and Canal client. A Canal server can connect to multiple MySQL instances, and maintains an event queue for each instance. Canal clients can then subscribe to theses queues and receive data changes. And the following is a quick start guide to get Canal going.

<!-- more -->

### Configure MySQL Master

`my.cnf`

```
server-id = 1
log_bin = /path/to/mysql-bin.log
binlog_format = ROW
```

```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT
ON *.* TO 'canal'@'127.0.0.1' IDENTIFIED BY 'canal';
```

### Startup Canal Server

### Write Canal Client

## Load into Data Warehouse

### Write to MySQL

### Write to Hive

* hive transaction
* join by primary key
* hbase

## Initial Target Table

* historical data - otter https://github.com/alibaba/otter/wiki/Manager%E9%85%8D%E7%BD%AE%E4%BB%8B%E7%BB%8D#%E8%87%AA%E5%AE%9A%E4%B9%89%E6%95%B0%E6%8D%AE%E5%90%8C%E6%AD%A5%E8%87%AA-%E7%94%B1-%E9%97%A8

## Canal High Availability

* canal HA
* master/slave

## References

* https://github.com/alibaba/canal/wiki (in Chinese)
* https://github.com/alibaba/otter/wiki (in Chinese)
* [Canal Project Introduction](https://docs.google.com/presentation/d/1MkszUPYRDkfVPz9IqOT1LLT5d9tuwde_WC8GZvjaDRg/edit?usp=sharing) (in Chinese)
* https://hortonworks.com/blog/four-step-strategy-incremental-updates-hive/

[1]: http://sqoop.apache.org/
[2]: https://github.com/alibaba/canal
