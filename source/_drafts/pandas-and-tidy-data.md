---
title: Pandas and Tidy Data
categories: [Big Data]
tags: [pandas, analytics, python]
---

In the paper [Tidy Data][1], [Dr. Wickham][2] proposed a specific form of data structure: each variable is a column, each observation is a row, and each type of observational unit is a table. He argued that with tidy data, data analysts can manipulate, model, and visualize data more easily and effectively. He lists five common data structures that are untidy, and demonstrates how to use [R language][3] to tidy them. In this article, we'll use Python and Pandas to achieve the same tidiness.

Source code and demo data can be found on GitHub ([link][4]), and readers are supposed to have Python environment installed, preferably with Anaconda and Spyder IDE.

## Column headers are values, not variable names

```python
import pandas as pd
df = pd.read_csv('data/pew.csv')
df.head(10)
```

![Religion and Income - Pew Forum](/images/tidy-data/pew.png)

Column names "<$10k", "$10-20k" are really income ranges that constitutes a variable. Variables are measurements of attributes, like height, weight, and in this case, income and religion. The values within the table form another variable, frequency. To make *each variable is a column*, we do the following transformation:

```python
df = df.set_index('religion')
df = df.stack()
df.index = df.index.rename('income', level=1)
df.name = 'frequency'
df = df.reset_index()
df.head(10)
```

![Religion and Income - Tidy](/images/tidy-data/pew-tidy.png)

<!-- more -->

Here use the [stack / unstack][5] feature of Pandas MultiIndex objects. `stack()` will use the column names to form a second level of index, then we do some proper naming and use `reset_index()` to flatten the table. In line 4 `df` is actually a Series, since Pandas will automatically convert from a single-column DataFrame.

Pandas provides another more commonly used method to do the transform, [`melt()`][6]. It accepts the following arguments:

* `frame`: the DataFrame to manipulate.
* `id_vars`: columns that stay put.
* `value_vars`: columns that will be transformed to a variable.
* `var_name`: name the newly added variable column.
* `value_name`: name the value column.

```python
df = pd.read_csv('data/pew.csv')
df = pd.melt(df, id_vars=['religion'], value_vars=list(df.columns)[1:],
             var_name='income', value_name='frequency')
df = df.sort_values(by='religion')
df.to_csv('data/pew-tidy.csv', index=False)
df.head(10)
```

This will give the same result. We'll use `melt()` method a lot in the following sections.

## Multiple variables stored in one column

## Variables are stored in both rows and columns

## Multiple types in one table

## One type in multiple tables

* Abstract
  * easy to manipulate, model, and visualize
  * specific structure
    * each variable is a column
    * each observation is a row
    * each type of obervational unit is a table
* Introduction
  * data tidying: structuring datasets to facilitate analysis
  * provide a standard way to organize data values within a dataset.
  * R packages: reshape, reshape2, plyr, ggplot2
* Defining tidy data
  * a dataset is a collection of values; value belongs to a variable and an observation; a variable contains all values that measure the same attribute across units; an observation contains all values measured on the same unit across atributes.
  * Codd's 3rd normal form
* Tidying messy datasets
  * common problems
    * column headers are values
    * multiple variables are stored in one column
    * variables are stored in both rows and columns
    * multiple types of observational units are stored in the same table
    * a single observational unit is stored in multiple tables
* Tidy tools
  * manipulation: filter, transform, aggregate, sort
  * visualization
  * modeling


## References

* https://tomaugspurger.github.io/modern-5-tidy.html
* https://hackernoon.com/reshaping-data-in-python-fa27dda2ff77
* http://www.jeannicholashould.com/tidy-data-in-python.html

[1]: https://www.jstatsoft.org/article/view/v059i10
[2]: https://en.wikipedia.org/wiki/Hadley_Wickham
[3]: https://github.com/hadley/tidy-data/
[4]: https://github.com/jizhang/pandas-tidy-data
[5]: https://pandas.pydata.org/pandas-docs/stable/reshaping.html
[6]: https://pandas.pydata.org/pandas-docs/stable/generated/pandas.melt.html
