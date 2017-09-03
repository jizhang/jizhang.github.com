---
title: Hive Window and Analytical Functions
categories: [Big Data]
tags: [hive, analytics, sql]
---

SQL is one of the major tools of data analysis. It provides filtering, transforming and aggregation functionalities, and we can use it to process big volume of data with the help of Hive and Hadoop. However, legacy SQL does not support operations like grouped ranking and moving average, because the `GROUP BY` clause can only produce one aggregation result for each group, but not for each row. Fortunately, with the new SQL standard coming, we can use the `WINDOW` clause to compute aggregations on a set of rows and return the result for each row.

![Moving Average](/images/hive-window/window-stock.png)

For instance, if we want to calculate the two-day moving average for each stock, we can write the following query:

```sql
SELECT
  `date`, `stock`, `close`
  ,AVG(`close`) OVER `w` AS `mavg`
FROM `t_stock`
WINDOW `w` AS (PARTITION BY `stock` ORDER BY `date`
               ROWS BETWEEN 1 PRECEDING AND CURRENT ROW)
```

`OVER`, `WINDOW` and `ROWS BETWEEN AND` are all newly added SQL keywords to support windowing operations. In this query, `PARTITION BY` and `ORDER BY` works like `GROUP BY` and `ORDER BY` after the `WHERE` caluse, except it doesn't collapse the rows, but only divides them into non-overlapping partitions to work on. `ROWS BETWEEN AND` here constructs a **window frame**. In this case, each frame contains the previous row and current row. We'll discuss more on frames later. Finally, `AVG` is a window function that computes results on each frame. Note that `WINDOW` clause can also be directly appended to window function:

```sql
SELECT AVG(`close`) OVER (PARTITION BY `stock`) AS `mavg` FROM `t_stock`;
```

<!-- more -->

## Window Query Concepts

![Concepts](/images/hive-window/concepts.png)

[Source][1]

SQL window query introduces three concepts, namely window partition, window frame and window function.

`PARTITION` clause divides result set into **window partitions** by one or more columns, and the rows whithin can be optionally sorted by one or more columns. If there's not `PARTITION BY`, the entrie result set is treated as a single partition; if there's not `ORDER BY`, window frames cannot be defined, and all rows within the partition constitudes a single frame.

**Window frame** selects rows from partition for window function to work on. There're two ways of defining frame in Hive, `ROWS` AND `RANGE`. For both types, we defines the upper bound and lower bound. For instance, `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` selects rows from the beginning of the partition to the current row; `SUM(close) RANGE BETWEEN 100 PRECEIDING AND 200 FOLLOWING` selects rows by the *distance* from the current row's value. Say current `close` is `200`, and this frame will includes rows whose `close` values range from `100` to `400`, within the partition. All possible combinations of frame definitions are listed as follows, and the default definition is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

```text
(ROWS | RANGE) BETWEEN (UNBOUNDED | [num]) PRECEDING AND ([num] PRECEDING | CURRENT ROW | (UNBOUNDED | [num]) FOLLOWING)
(ROWS | RANGE) BETWEEN CURRENT ROW AND (CURRENT ROW | (UNBOUNDED | [num]) FOLLOWING)
(ROWS | RANGE) BETWEEN [num] FOLLOWING AND (UNBOUNDED | [num]) FOLLOWING
```

All **window functions** compute results on the current frame. Hive supports the following functions:

* `FIRST_VALUE(col)`, `LAST_VALUE(col)` returns the column value of first / last row within the frame;
* `LEAD(col, n)`, `LAG(col, n)` returns the column value of n-th row before / after current row;
* `RANK()`, `ROW_NUMBER()` assigns a sequence of the current row within the frame. The difference is `RANK()` will contain duplicate if there're identical values.
* `COUNT()`, `SUM(col)`, `MIN(col)` works as usual.

## Hive Query Examples

### Top K

First let's simulate some data of employee incomes in Hive:

```sql
CREATE t_employee (
  id INT
  ,emp_name VARCHAR(20)
  ,dep_name VARCHAR(20)
  ,salary DECIMAL(7, 2)
  ,age DECIMAL(3, 0)
);

INSERT INTO t_employee VALUES
( 1,  'Matthew', 'Management',  4500, 55),
( 2,  'Olivia',  'Management',  4400, 61),
( 3,  'Grace',   'Management',  4000, 42),
( 4,  'Jim',     'Production',  3700, 35),
( 5,  'Alice',   'Production',  3500, 24),
( 6,  'Michael', 'Production',  3600, 28),
( 7,  'Tom',     'Production',  3800, 35),
( 8,  'Kevin',   'Production',  4000, 52),
( 9,  'Elvis',   'Service',     4100, 40),
(10,  'Sophia',  'Sales',       4300, 36),
(11,  'Samantha','Sales',       4100, 38);
```

We can use the `RANK()` function to find out who earns the most within each department:

```sql
SELECT dep_name, emp_name, salary
FROM (
  SELECT
    dep_name, emp_name, salary
    ,RANK() OVER (PARTITION BY dep_name ORDER BY salary DESC) AS rnk
  FROM t_employee
) a
where rnk = 1;
```

Normally when there's duplicates, `RANK()` returns the same value for each row and *skip* the next sequence number. Use `DENSE_RANK()` if you want consecutive ranks.

### Cumulative Distribution

We can calculate the cumulative distribution of salaries among all departments. For example, salary `4000`'s cumulative distribution is `0.55`, which means 55% people's salaries are less or equal to `4000`. To calculate this, we first count the frequencies of every salary, and do a cumulative summing:

```sql
SELECT
  salary
  ,SUM(cnt) OVER (ORDER BY salary)
  / SUM(cnt) OVER (ORDER BY salary ROWS BETWEEN UNBOUNDED PRECEDING
                   AND UNBOUNDED FOLLOWING)
FROM (
  SELECT salary, count(*) AS cnt
  FROM t_employee
  GROUP BY salary
) a;
```

This can also be done with Hive's `CUME_DIST()` window function. There's another `PERCENT_RANK()` function, which computes the rank of the salary as percentage.

```sql
SELECT
  salary
  ,CUME_DIST() OVER (ORDER BY salary) AS pct_cum
  ,PERCENT_RANK() OVER (ORDER BY salary) AS pct_rank
FROM t_employee;
```

![Cumulative Distribution](/images/hive-window/employee-pct.png)

### Clickstream Sessionization

We can divide click events into different sessions by setting a *timeout*, in this case 30 minutes, and assign an id to each session:

![Click Stream](/images/hive-window/clickstream.png)

First, in subquery `b`, we use the `LAG(col)` function to calculate the time difference between current row and previous row, and if it's more than 30 minutes, a new session is marked. Then we do a cumulative sum of the `new_session` field so that each session will get an incremental sequence.

```sql
SELECT
  ipaddress, clicktime
  ,SUM(IF(new_session, 1, 0)) OVER x + 1 AS sessionid
FROM (
  SELECT
    ipaddress, clicktime, ts
    ,ts - LAG(ts) OVER w > 1800 AS new_session
  FROM (
    SELECT *, UNIX_TIMESTAMP(clicktime) AS ts
    FROM t_clickstream
  ) a
  WINDOW w AS (PARTITION BY ipaddress ORDER BY ts)
) b
WINDOW x AS (PARTITION BY ipaddress ORDER BY ts);
```

## Implementation Detail

Partitioned Table Function

## References

* https://cwiki.apache.org/confluence/display/Hive/LanguageManual+WindowingAndAnalytics
* https://issues.apache.org/jira/secure/attachment/12575830/WindowingSpecification.pdf
* https://www.slideshare.net/Hadoop_Summit/analytical-queries-with-hive
* https://en.wikibooks.org/wiki/Structured_Query_Language/Window_functions
* https://databricks.com/blog/2015/07/15/introducing-window-functions-in-spark-sql.html

[1]: https://en.wikibooks.org/wiki/Structured_Query_Language/Window_functions
