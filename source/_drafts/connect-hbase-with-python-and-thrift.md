---
title: Connect HBase with Python and Thrift
categories: Big Data
tags: [python, hbase, thrift]
date: 2018-04-22 16:44:12
---

[Apache HBase][1] is a key-value store in Hadoop ecosystem. It is based on HDFS, and can provide high performance data access on large amount of volume. HBase is written in Java, and has native support for Java clients. But with the help of Thrift and various language bindings, we can access HBase in web services quite easily. This article will describe how to read and write HBase table with Python and Thrift.

![](/images/hbase.png)

## Generate Thrift Class

For anyone who is new to [Apache Thrift][2], it provides an IDL (Interface Description Language) to let you describe your service methods and data types and then transform them into different languages. For instance, a Thrift type definition like this:

```thrift
struct TColumn {
  1: required binary family,
  2: optional binary qualifier,
  3: optional i64 timestamp
}
```

Will be transformed into the following Python code:

```python
class TColumn(object):
    def __init__(self, family=None, qualifier=None, timestamp=None,):
        self.family = family
        self.qualifier = qualifier
        self.timestamp = timestamp

    def read(self, iprot):
        iprot.readStructBegin()
        while True:
            (fname, ftype, fid) = iprot.readFieldBegin()
            # ...

    def write(self, oprot):
        oprot.writeStructBegin('TColumn')
        # ...
```

<!-- more -->

### HBase Thrift vs Thrift2

HBase provides [two versions][3] of Thrift IDL files, and they have two main differences.

First, `thrift2` mimics the data types and methods from HBase Java API, which could be more intuitive to use. For instance, constructing a `Get` operation in Java is:

```java
Get get = new Get(Bytes.toBytes("rowkey"));
get.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("col1"));
get.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("col2"));
```

In `thrift2`, there is a corresponding `TGet` type:

```python
tget = TGet(
    row='rowkey',
    columns=[
        TColumn(family='cf', qualifier='col1'),
        TColumn(family='cf', qualifier='col2'),
    ]
)
```

While in `thrift`, we directly invoke one of the `get` methods:

```python
client.getRowWithColumns(
    tableName='tbl',
    row='rowkey',
    columns=['cf:col1', 'cf:col2'],
    attributes=None
)
```

The second difference is that `thrift2` lacks the administration interfaces, like `createTable`, `majorCompact`, etc. Currently these APIs are still under development, so if you need to use them via Thrift, you will have to fall back to version one.

After deciding which version we use, now we can download the `hbase.thrift` file, and generate Python code from it. One note on Thrift version though. Since we will use Python 3.x, which is supported by Thrift 0.10 onwards, so make sure you install the right version. Execute the following command, and you will get several Python files.

```bash
$ thrift -gen py hbase.thrift
$ find gen-py
gen-py/hbase/__init__.py
gen-py/hbase/constants.py
gen-py/hbase/THBaseService.py
gen-py/hbase/ttypes.py
```

## Run HBase in Standalone Mode

In case you do not have a running HBase service to test against, you can follow the quick start guide ([link][4]) to download the binaries, do some minor configuration, and then execute the following commands to start a standalone HBase server as well as the Thrift2 server.

```bash
bin/start-hbase.sh
bin/hbase-daemon.sh start thrift2
bin/hbase shell
```

Then in the HBase shell, we create a test table and read / write some data.

```ruby
> create "tsdata", NAME => "cf"
> put "tsdata", "sys.cpu.user:20180421:192.168.1.1", "cf:1015", "0.28"
> get "tsdata", "sys.cpu.user:20180421:192.168.1.1"
COLUMN                                        CELL
 cf:1015                                      timestamp=1524277135973, value=0.28
1 row(s) in 0.0330 seconds
```

## Connect to HBase via Thrift2

Here is the boilerplate of making a connection to HBase Thrift server. Note that Thrift client is not thread-safe, and it does neither provide connection pooling facility. You may choose to connect on every request, which is actually fast enough, or maintain a pool of connections yourself.

```python
from thrift.transport import TSocket
from thrift.protocol import TBinaryProtocol
from thrift.transport import TTransport
from hbase import THBaseService

transport = TTransport.TBufferedTransport(TSocket.TSocket('127.0.0.1', 9090))
protocol = TBinaryProtocol.TBinaryProtocolAccelerated(transport)
client = THBaseService.Client(protocol)
transport.open()
# perform some operations with "client"
transport.close()
```

We can test the connection with some basic operations:

```python
from hbase.ttypes import TPut, TColumnValue, TGet
tput = TPut(
    row='sys.cpu.user:20180421:192.168.1.1',
    columnValues=[
        TColumnValue(family='cf', qualifier='1015', value='0.28'),
    ]
)
client.put('tsdata', tput)

tget = TGet(row='sys.cpu.user:20180421:192.168.1.1')
tresult = client.get('tsdata', tget)
for col in tresult.columnValues:
    print(col.qualifier, '=', col.value)
```

## Thrift2 Data Types and Methods Overview

For a full list of the available APIs, one can directly look into `hbase.thrift` or `hbase/THBaseService.py` files. Following is an abridged table of those data types and methods.

### Data Types

| Class | Description | Example |
| --- | --- | --- |
| TColumn | Represents a column family or a single column. | TColumn(family='cf', qualifier='gender') |
| TColumnValue | Column and its value. | TColumnValue(family='cf', qualifier='gender', value='male') |
| TResult | Query result, a single row. `row` attribute would be `None` if no result is found. | TResult(row='employee_001', columnValues=[TColumnValue]) |
| TGet | Query a single row. | TGet(row='employee_001', columns=[TColumn]) |
| TPut | Mutate a single row. | TPut(row='employee_001', columnValues=[TColumnValue]) |
| TDelete | Delete an entire row or only some columns. | TDelete(row='employee_001', columns=[TColumn]) |
| TScan | Scan for multiple rows and columns. | See below. |

### THBaseService Methods

| Method Signature | Description |
| --- | --- |
| get(table: str, tget: TGet) -> TResult | Query a single row. |
| getMultiple(table: str, tgets: List[TGet]) -> List[TResult] | Query multiple rows. |
| put(table: str, tput: TPut) -> None | Mutate a row. |
| putMultiple(table: str, tputs: List[TPut]) -> None | Mutate multiple rows. |
| deleteSingle(table: str, tdelete: TDelete) -> None | Delete a row. |
| deleteMultiple(table: str, tdeletes: List[TDelete]) -> None | Delete multiple rows. |
| openScanner(table: str, tscan: TScan) -> int | Open a scanner, returns scannerId. |
| getScannerRows(scannerId: int, numRows: int) -> List[TResult] | Get scanner rows. |
| closeScanner(scannerId: int) -> None | Close a scanner. |
| getScannerResults(table: str, tscan: TScan, numRows: int) -> List[TResult] | A convenient method to get scan results. |

### Scan Operation Example

I wrote some example codes on GitHub ([link][5]), and the following is how a `Scan` operation is made.

```python
scanner_id = client.openScanner(
    table='tsdata',
    tscan=TScan(
        startRow='sys.cpu.user:20180421',
        stopRow='sys.cpu.user:20180422',
        columns=[TColumn('cf', '1015')]
    )
)
try:
    num_rows = 10
    while True:
        tresults = client.getScannerRows(scanner_id, num_rows)
        for tresult in tresults:
            print(tresult)
        if len(tresults) < num_rows:
            break
finally:
    client.closeScanner(scanner_id)
```

## Thrift Server High Availability

There are several solutions to eliminate the single point of failure of Thrift server. You can either (1) randomly select a server address on the client-side, and fall back to others if failure is detected, (2) setup a proxy facility to load balance the TCP connections, or (3) run individual Thrift server on every client machine, and let client code connects the local Thrift server. Usually we use the second approach, so you may consult your system administrator on that topic.

![](/images/hbase-thrift-ha.png)

## References

* https://blog.cloudera.com/blog/2013/09/how-to-use-the-hbase-thrift-interface-part-1/
* https://thrift.apache.org/tutorial/py
* https://yq.aliyun.com/articles/88299
* http://opentsdb.net/docs/build/html/user_guide/backends/hbase.html


[1]: https://hbase.apache.org/
[2]: https://thrift.apache.org/
[3]: https://github.com/apache/hbase/tree/master/hbase-thrift/src/main/resources/org/apache/hadoop/hbase
[4]: https://hbase.apache.org/book.html#quickstart
[5]: https://github.com/jizhang/python-hbase
