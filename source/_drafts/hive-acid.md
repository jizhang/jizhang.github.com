---
title: Understanding Hive ACID Transactional Table
tags: [hive, hadoop]
categories: Big Data
---

[Apache Hive][1] introduced transactions since version 0.13 to fully support ACID semantics on Hive table, including INSERT/UPDATE/DELETE/MERGE statements, streaming data ingestion, etc. In Hive 3.0, this feature is further improved by optimizing the underlying data file structure, reducing constraints on table scheme, and supporting predicate push down and vectorized query. Examples and setup can be found on [Hive wiki][2] and other [tutorials][3], while this article will focus on how transactional table is saved on HDFS, and take a closer look at the read-write process.

![]()

## File Structure

### Insert Data

```sql
CREATE TABLE employee (
    id int
    ,name string
    ,salary int
)
STORED AS ORC
TBLPROPERTIES ('transactional' = 'true');

INSERT INTO employee VALUES
(1, 'Jerry', 5000),
(2, 'Tom',   8000),
(3, 'Kate',  6000);
```

An INSERT statement is executed in a single transaction. It will create a `delta` directory containing information about this transaction and its data.

```text
/user/hive/warehouse/employee/delta_0000001_0000001_0000
/user/hive/warehouse/employee/delta_0000001_0000001_0000/_orc_acid_version
/user/hive/warehouse/employee/delta_0000001_0000001_0000/bucket_00000
```

The schema of this folder's name is `delta_minWID_maxWID_stmtID`, i.e. "delta" prefix, transactional writes' range (minimum and maximum write ID), and statement ID. In detail:

* All INSERT statements will create a `delta` directory. UPDATE statement will also create `delta` directory right after a `delete` directory. `delete` directory is prefixed with "delete_delta".
* Hive will assign a globally unique ID for every transaction, both read and write. For transactional writes like INSERT and DELETE, it will also assign a table-wise unique ID, a.k.a. a write ID. The write ID range will be encoded in the `delta` and `delete` directory names.
* Statement ID is used when multiple writes into the same table happen in one transaction.

<!-- more -->

For its content, `_orc_acid_version` always contains "2", indicating this directory is in ACID version 2 format. The main difference is that UPDATE now uses split-update technique to support predicate push down and other features ([HIVE-14035][4]). `bucket_00000` is the inserted records. Since this table is not bucketed, there is only one file, and it is in [ORC][5] format. We can take a look at its content with [orc-tools][6]:

```text
$ orc-tools data bucket_00000
{"operation":0,"originalTransaction":1,"bucket":536870912,"rowId":0,"currentTransaction":1,"row":{"id":1,"name":"Jerry","salary":5000}}
{"operation":0,"originalTransaction":1,"bucket":536870912,"rowId":1,"currentTransaction":1,"row":{"id":2,"name":"Tom","salary":8000}}
{"operation":0,"originalTransaction":1,"bucket":536870912,"rowId":2,"currentTransaction":1,"row":{"id":3,"name":"Kate","salary":6000}}
```

The file content is displayed in JSON, row-wise. We can see the actual data is in `row`, while other keys work for transaction mechanism:

* `operation` 0 means INSERT, 1 UPDATE, and 2 DELETE. UPDATE will not appear because of the split-update technique mentioned above.
* `originalTransaction` is the previous write ID. For INSERT, it is the same as `currentTransaction`. For DELETE, it is the write ID when this record is first created.
* `bucket` is a 32-bit integer defined by BucketCodec class. Their meanings are:
    * bit 1-3: bucket codec version, currently `001`.
    * bit 4: reserved for future.
    * bit 5-16: the bucket ID, 0-based. This ID is determined by CLUSTERED BY columns and number of buckets. It matches the `bucket_N` prefixed files.
    * bit 17-20: reserved for future.
    * bit 21-32: statement ID.
    * For instance, the binary form of `536936448` is `00100000000000010000000000000000`, showing it is a version 1 codec, and bucket ID is 1.
* `rowId` TODO
* `currentTransaction` is the current write ID.
* `row` contains the actual data. For DELETE, `row` will be null.

These information can also be viewed by the `row__id` virtual column:

```sql
SELECT row__id, id, name, salary FROM employee;
```

Output:

```text
{"writeid":1,"bucketid":536870912,"rowid":0}    1       Jerry   5000
{"writeid":1,"bucketid":536870912,"rowid":1}    2       Tom     8000
{"writeid":1,"bucketid":536870912,"rowid":2}    3       Kate    6000
```

### Update Data

```sql
UPDATE employee SET salary = 7000 WHERE id = 2;
```

This statement will first run a query to find out the `row__id` of the updating records, and then create a `delete` directory a long with a `delta` directory:

```text
/user/hive/warehouse/employee/delta_0000001_0000001_0000/bucket_00000
/user/hive/warehouse/employee/delete_delta_0000002_0000002_0000/bucket_00000
/user/hive/warehouse/employee/delta_0000002_0000002_0000/bucket_00000
```

Content of `delete_delta_0000002_0000002_0000/bucket_00000`:

```text
{"operation":2,"originalTransaction":1,"bucket":536870912,"rowId":1,"currentTransaction":2,"row":null}
```

Content of `delta_0000002_0000002_0000/bucket_00000`:

```text
{"operation":0,"originalTransaction":2,"bucket":536870912,"rowId":0,"currentTransaction":2,"row":{"id":2,"name":"Tom","salary":7000}}
```

DELETE statement is omitted here, since its process is similar to UPDATE, i.e. find the record and then generate only `delete` directory.

### Merge Statement

### Partitioned and Bucketed Table

## Merge on Read



## Compaction



## Transaction Manager



## References



[1]: http://hive.apache.org/
[2]: https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions
[3]: https://hortonworks.com/tutorial/using-hive-acid-transactions-to-insert-update-and-delete-data/
[4]: https://jira.apache.org/jira/browse/HIVE-14035
[5]: https://orc.apache.org/
[6]: https://orc.apache.org/docs/java-tools.html


try acid operation with local hive
analyze the file change in different statements (insert, update, delete, merge)
    https://community.hortonworks.com/articles/97113/hive-acid-merge-by-example.html
read AcidInputFormat source code
    orc file format
vectorized input reader
write output format
stream api
compaction
transaction manager

## file names

base_0000001/bucket_00000
base_(writer_id)

delete_delta_0000002_0000002_0000

delta_0000002_0000002_0000

delta files are soreted by key

row__id
    writeid: transaction id
    bucketid
    rowid: index in this transaction-bucket

read split
    base/bN + delete_delta
    delta/bN + delete_delta
compact split: base/bN + delta/bN + delete_delta/bALL?

## compaction

minor
major

slowly-changing dimensions

## References

https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions


https://www.slideshare.net/Hadoop_Summit/transactional-operations-in-apache-hive-present-and-future-102803358

https://hortonworks.com/tutorial/using-hive-acid-transactions-to-insert-update-and-delete-data/
https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.1.0/using-hiveql/content/hive_3_internals.html
https://www.slideshare.net/Hadoop_Summit/what-is-new-in-apache-hive-30

https://hive.apache.org/javadocs/r3.1.1/api/org/apache/hadoop/hive/ql/io/AcidInputFormat.html


https://orc.apache.org/docs/acid.html

## QUESTION

acid insert vs normal insert?
why orc?
    ORC format can support large batches of mutations in a transaction https://cwiki.apache.org/confluence/display/Hive/HCatalog+Streaming+Mutation+API
primary key, unique key?
snapshot isolation?
delta write id range? - generated by minor compaction
current write id vs original write id?
streaming update? - query for row__id first
孙将军，细节：
    binlog 直接分桶后写入文件，没有其他处理吗？比如按主键排序之类
    时效性如何，是用 fluentd 实时流进去的吗？
    读取的时候是怎样合并的？删除、更新的记录是怎样读取的？
    会不会定期去清理文件，类似 hbase compaction
    源码能参考一下吗
why acid?
how is bucket id generated? bucketed/partitioned table vs. non-bucketed

MISC
turn on log

NOTES


CREATE DATABASE merge_data;

CREATE TABLE merge_data.transactions(
  ID int,
  TranValue string,
  last_update_user string)
PARTITIONED BY (tran_date string)
CLUSTERED BY (ID) into 5 buckets
STORED AS ORC TBLPROPERTIES ('transactional'='true');

CREATE TABLE merge_data.merge_source(
  ID int,
  TranValue string,
  tran_date string)
STORED AS ORC;

INSERT INTO merge_data.transactions PARTITION (tran_date) VALUES
(1, 'value_01', 'creation', '20170410'),
(2, 'value_02', 'creation', '20170410'),
(3, 'value_03', 'creation', '20170410'),
(4, 'value_04', 'creation', '20170410'),
(5, 'value_05', 'creation', '20170413'),
(6, 'value_06', 'creation', '20170413'),
(7, 'value_07', 'creation', '20170413'),
(8, 'value_08', 'creation', '20170413'),
(9, 'value_09', 'creation', '20170413'),
(10, 'value_10','creation', '20170413');

INSERT INTO merge_data.merge_source VALUES
(1, 'value_01', '20170410'),
(4, NULL, '20170410'),
(7, 'value_77777', '20170413'),
(8, NULL, '20170413'),
(8, 'value_08', '20170415'),
(11, 'value_11', '20170415');

MERGE INTO merge_data.transactions AS T
USING merge_data.merge_source AS S
ON T.ID = S.ID and T.tran_date = S.tran_date
WHEN MATCHED AND (T.TranValue != S.TranValue AND S.TranValue IS NOT NULL) THEN UPDATE SET TranValue = S.TranValue, last_update_user = 'merge_update'
WHEN MATCHED AND S.TranValue IS NULL THEN DELETE
WHEN NOT MATCHED THEN INSERT VALUES (S.ID, S.TranValue, 'merge_insert', S.tran_date);

select * from transactions order by id;



CREATE TABLE merge_data.transactions(
  ID int,
  TranValue string,
  last_update_user string,
  tran_date string)
STORED AS ORC TBLPROPERTIES ('transactional'='true');


INSERT INTO merge_data.transactions VALUES
(1, 'value_01', 'creation', '20170410'),
(2, 'value_02', 'creation', '20170410'),
(3, 'value_03', 'creation', '20170410'),
(4, 'value_04', 'creation', '20170410'),
(5, 'value_05', 'creation', '20170413'),
(6, 'value_06', 'creation', '20170413'),
(7, 'value_07', 'creation', '20170413'),
(8, 'value_08', 'creation', '20170413'),
(9, 'value_09', 'creation', '20170413'),
(10, 'value_10','creation', '20170413');
