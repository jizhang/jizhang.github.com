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

Solution B: Increment by Number of Tasks
----------------------------------------

Inspired by a [mailing list][3] that is quite hard to find, which is inspired by MySQL master-master setup (with auto\_increment\_increment and auto\_increment\_offset), there's a brilliant way to generate a globally unique integer id across mappers or reducers. Let's take mapper for example:

```java
public static class JobMapper extends Mapper<LongWritable, Text, LongWritable, Text> {

    private long id;
    private int increment;

    @Override
    protected void setup(Context context) throws IOException,
            InterruptedException {

        super.setup(context);

        id = context.getTaskAttemptID().getTaskID().getId();
        increment = context.getConfiguration().getInt("mapred.map.tasks", 0);
        if (increment == 0) {
            throw new IllegalArgumentException("mapred.map.tasks is zero");
        }
    }

    @Override
    protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {

        id += increment;
        context.write(new LongWritable(id),
                new Text(String.format("%d, %s", key.get(), value.toString())));
    }

}
```

The basic idea is simple:

1. Set the initial id to current tasks's id.
2. When mapping each row, increment the id by the number of tasks.

It's also applicable to reducers.

Solution C: Sorted Auto-increment Id
------------------------------------

Here's a real senario: we have several log files pulled from different machines, and we want to identify each row by an auto-increment id, and they should be in time sequence order.

We know Hadoop has a sort phase, so we can use timestamp as the mapper output key, and the framework will do the trick. But the sorting thing happends in one reducer (partition, in fact), so when using multiple reducer tasks, the result is not in total order. To achieve this, we can use the [TotalOrderPartitioner][4].

How about the incremental id? Even though the outputs are in total order, Solution B is not applicable here. So we take another approach: seperate the job in two phases, use the reducer to do sorting *and* counting, then use the second mapper to generate the id.

Here's what we gonna do:

1. Use TotalOrderPartitioner, and set the sampler class.
2. Parse logs in mapper A, use time as the output key.
3. Let the framework do partitioning and sorting.
4. Count records in reducer, write it with [MultipleOutput][5].
5. In mapper B, use count as offset, and increment by 1.


[1]: http://dev.mysql.com/doc/refman/5.1/en/example-auto-increment.html
[2]: http://docs.mongodb.org/manual/tutorial/create-an-auto-incrementing-field/
[3]: http://mail-archives.apache.org/mod_mbox/hadoop-common-user/200904.mbox/%3C49E13557.7090504@domaintools.com%3E
[4]: http://hadoop.apache.org/docs/r1.0.4/api/org/apache/hadoop/mapred/lib/TotalOrderPartitioner.html
[5]: http://hadoop.apache.org/docs/r1.0.4/api/org/apache/hadoop/mapreduce/lib/output/MultipleOutputs.html
