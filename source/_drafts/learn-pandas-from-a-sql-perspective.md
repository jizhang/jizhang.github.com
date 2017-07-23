---
title: Learn Pandas from a SQL Perspective
tags: [pandas, sql, analytics, python]
categories: [Big Data]
---

[Pandas](http://pandas.pydata.org/) is a widely used data processing tool for Python. Along with NumPy and Matplotlib, it provides in-memory high-performance data munging, analyzing, and visualization capabilities. Although Python is an easy-to-learn programming language, it still takes time to learn Pandas APIs and the idiomatic usages. For data engineer and analysts, SQL is the de-facto standard language of data queries. This article will provide examples of how some common SQL queires can be rewritten with Pandas.

The installation and basic concepts of Pandas is not covered in this post. One can check out the offical documentation, or read the book [Python for Data Analysis][1]. And I recommend using the [Anaconda][2] Python distribution, with [Spyder][3] IDE included. Before diving into the codes, please import Pandas and NumPy as follows:

```python
import pandas as pd
import numpy as np
```

## `FROM` - Load Data into Memory

First of all, let's read some data into the workspace (memory). Pandas supports a variety of formats, one of them is CSV. Take the following flight delay dataset for example ([link](/uploads/flights.csv)):

```csv
date,delay,distance,origin,destination
02221605,3,358,BUR,SMF
01022100,-5,239,HOU,DAL
03210808,6,288,BWI,ALB
```

We can use `pd.read_csv` to load this file:

```python
df = pd.read_csv('flights.csv', dtype={'date': str})
df.head()
```

This statement will load `flights.csv` file into memory, use first line as column names, and try to figure our each column's type. Since the `date` column is in `%m%d%H%M` format, we don't want to lose the initial `0` in month, so we pass an explict `dtype` for it, indicating that this column should stay unparsed.

<!-- more -->

 `df.head` is a function to peek the dataset. It accepts a single parameter to limit the rows, much like `LIMIT` caluse. To perform a `LIMIT 10, 100`, use `df.iloc[10:100]`. Besides, IPython defaults to show only 60 rows, but we can increase this limit by:

```python
pd.options.display.max_rows = 100
df.iloc[10:100]
```

Another common loading technique is reading from database. Pandas also has built-in support:

```python
conn = pymysql.connect(host='localhost', user='root')
df = pd.read_sql("""
select `date`, `delay`, `distance`, `origin`, `destination`
from flights limit 1000
""", conn)
```

To save DataFrame into file or database, use `pd.to_csv` and `pd.to_sql` respectively.

## `SELECT` - Column Projection

The `SELECT` clause in SQL is used to perform column projection and data transformation.

```python
df['date'] # SELECT `date`
df[['date', 'delay']] # SELECT `date`, `delay`
df.loc[10:100, ['date', 'delay']] # SELECT `date, `delay` LIMIT 10, 100
```

SQL provides various functions to transform data, most of them can be replaced by Pandas, or you can simply write one with Python. Here I'll choose some commonly used functions to illustrate.

### String Functions

Pandas string functions can be invoked by DataFrame and Series' `str` attribute, e.g. `df['origin'].str.lower()`.

```python
# SELECT CONCAT(origin, ' to ', destination)
df['origin'].str.cat(df['destination'], sep=' to ')

df['origin'].str.strip() # TRIM(origin)
df['origin'].str.len() # LENGTH(origin)
df['origin'].str.replace('a', 'b') # REPLACE(origin, 'a', 'b')

# SELECT SUBSTRING(origin, 1, 1)
df['origin'].str[0:1] # use Python string indexing

# SELECT SUBSTRING_INDEX(domain, '.', 2)
# www.example.com -> www.example
df['domain'].str.split('.').str[:2].str.join('.')
df['domain'].str.extract(r'^([^.]+\.[^.]+)')
```

Pandas also has a feature called broadcast behaviour, i.e. perform operations between lower dimensional data (or scalar value) with higher dimensional data. For instances:

```python
df['full_date'] = '2001' + df['date'] # CONCAT('2001', `date`)
df['delay'] / 60
df['delay'].div(60) # same as above
```

There're many other string functions that Pandas support out-of-the-box, and they are quite different, thus more powerful than SQL. For a complete list please check the [Working with Text Data][4] doc.

### Date Functions

str_to_date, date_format, dateadd, timestamp, hour

```python
# SELECT STR_TO_DATE(full_date, '%Y%m%d%H%i%s') AS `datetime`
df['datetime'] = pd.to_datetime(df['full_date'], format='%Y%m%d%H%M%S')

# SELECT DATE_FORMAT(`datetime`, '%Y-%m-%d')
df['datetime'].dt.strftime('%Y-%m-%d')

df['datetime'].dt
```

`dt` is a property of datetime/period like Series.


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
[4]:https://pandas.pydata.org/pandas-docs/stable/text.html
