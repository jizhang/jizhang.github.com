---
title: MPP
categories: Big Data
tags: []
---

<!-- more -->

## References

http://prestodb.rocks/internals/the-fundamentals-data-distribution/
https://cwiki.apache.org/confluence/display/IMPALA/Impala+Reading+List
https://qr.ae/TWtXLq
    impala has memory limit, suitable for small table joins
    hive can join very large tables

http://pages.cs.wisc.edu/~floratou/SQLOnHadoop.pdf
    performance comparison
    when parquet is faster than text?

paper: impala http://cidrdb.org/cidr2015/Papers/CIDR15_Paper28.pdf
    join strategy: broadcast, partitioned
    disributed local pre-aggregation + single-node merge aggregation
    query plan
        two-phase
        fragment
    code generation, 5x more performance gain
        c++
        instruction count, memory overhead
        runtime information, e.g. a function that handles only integers will be faster than that handles general types
        virtual function calls - inlining
    volcano-style processing with exchange operators
    io
        short circuit local reads, almost full disk bandwidth
        HDFS caching
        io manager
    file formats
        compression
    resource manager
        admission control, de-centralized, receive from statestore asynchronously, simple throttling mechanism

paper: dremel https://research.google.com/pubs/archive/36632.pdf
    root server, intermediate server, leaf server
        multi-level serving tree
    tablet, slot


paper: presto

https://www.cloudera.com/documentation/enterprise/6/6.1/topics/impala_performance.html

QUESTIONS
mpp vs dag
join implementation difference
cache
unit of resource, cpu, memory, etc, for executing multiple queries at the same time
data locality
semi-join, anti-join
mysql execution engine? parallel?
parquest vs text in Hive?
impala vs drill vs presto
vs hive tez + llap
fault tolerance
