---
title: Learn Pandas from a SQL Perspective
tags: [pandas, sql, analytics, python]
categories: [Big Data]
---

[Pandas](http://pandas.pydata.org/) is a widely used data processing tool for Python. Along with NumPy and Matplotlib, it provides in-memory high-performance data munging, analyzing, and visualization capabilities. Although Python is an easy-to-learn programming language, it still takes time to learn Pandas APIs and the idiomatic usages. For data engineer and analysts, SQL is the de-facto standard language of data queries. This article will provide examples of how some common SQL queires can be rewritten with Pandas.

The installation and basic concepts of Pandas is not covered in this post. One can check out the offical documentation, or the read the book [Python for Data Analysis][1]. And I recommend using the [Anaconda][2] Python distribution, with [Spyder][3] IDE embedded. Before diving into the codes, please import Pandas and NumPy as follows:

```python
import pandas as pd
import numpy as np
```

## `FROM` - Load Data into Memory

First of all, let's read some data into the workspace (memory). Pandas support a lot of data formats, 

read from csv, mysql, excel, hive
save data

<!-- more -->

## `SELECT` - Column Projection

functions

## `WHERE` - Row Selection

## `GROUP BY` - Aggregation

## `ORDER BY` - Sorting Rows

## `JOIN` - Merge DateFrames

union

## `UPDATE` and `DELETE`

## pivot

## pandasql

## References

* https://pandas.pydata.org/pandas-docs/stable/comparison_with_sql.html
* http://www.gregreda.com/2013/01/23/translating-sql-to-pandas-part1/
* http://codingsight.com/pivot-tables-in-mysql/

[1]: https://www.amazon.com/Python-Data-Analysis-Wrangling-IPython/dp/1491957662/
[2]: https://www.continuum.io/downloads
[3]: https://pythonhosted.org/spyder/
