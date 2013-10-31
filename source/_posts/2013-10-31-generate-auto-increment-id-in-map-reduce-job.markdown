---
layout: post
title: "Generate Auto-increment Id in Map-reduce Job"
date: 2013-10-31 09:35
comments: true
categories: Notes
published: false
---

In DBMS world, it's easy to generate a unique, auto-increment id, using MySQL's [AUTO_INCREMENT attribute][1] on a primary key or MongoDB's [Counters Collection][2] pattern. But when it comes to a distributed, parallel processing framework, like Hadoop Map-reduce, it is not that straight forward. The best solution to identify every record in such framework is to use UUID. But when an integer id is required, it'll take some steps.

Solution A: Single Reducer
--------------------------

This is the most obvious and simple one, just use the following code to specify reducer numbers to 1:

```java
job.setNumReduceTasks(1);
```

And also obvious, there are several demerits:

1. All mappers output will be copied to one task tracker.
2. Only one process is working on shuffel & sort.
3. When producing output, there's also only one process.

The above is not a problem for small data sets, or at least small mapper outputs. And it is also the approach that Pig and Hive use when they need to perform a total sort. But when hitting a certain threshold, the sort and copy phase will become very slow and unacceptable.

<!-- more -->

[1]: http://dev.mysql.com/doc/refman/5.1/en/example-auto-increment.html
[2]: http://docs.mongodb.org/manual/tutorial/create-an-auto-incrementing-field/
