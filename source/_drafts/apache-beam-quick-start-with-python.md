---
title: Apache Beam Quick Start with Python
categories: [Big Data]
tags: [apache beam, python, mapreduce, stream processing]
---

[Apache Beam][1] is a big data processing standard created by Google in 2016. It provides unified DSL to process both batch and stream data, and can be executed on popular platforms like Spark, Flink, and of course Google's commercial product Dataflow. Beam's model is based on previous works known as [FlumeJava][2] and [Millwheel][3], and addresses solutions for data processing tasks like ETL, analysis, and [stream processing][4]. Currently it provides SDK in two languages, Java and Python. This article will introduce how to use Python to write Beam applications.

![Apache Beam Pipeline](/images/beam/arch.jpg)

## Installation

Apache Beam Python SDK requires Python 2.7.x. You can use [pyenv][5] to manage different Python versions, or compile from [source][6] (make sure you have SSL installed). And then you can install Beam SDK from PyPI, better in a virtual environment:

```
$ virtualenv venv --distribute
$ source venv/bin/activate
(venv) $ pip install apache-beam
```

<!-- more -->

## Wordcount Example

Wordcount is the de-facto "Hello World" in big data field, so let's take a look at how it's done with Beam:

```python
from __future__ import print_function
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
with beam.Pipeline(options=PipelineOptions()) as p:
    lines = p | 'Create' >> beam.Create(['cat dog', 'snake cat', 'dog'])
    counts = (
        lines
        | 'Split' >> (beam.FlatMap(lambda x: x.split(' '))
                      .with_output_types(unicode))
        | 'PairWithOne' >> beam.Map(lambda x: (x, 1))
        | 'GroupAndSum' >> beam.CombinePerKey(sum)
    )
    counts | 'Print' >> beam.ParDo(lambda (w, c): print('%s: %s' % (w, c)))
```

Run the script, you'll get the counts of difference words:

```
(venv) $ python wordcount.py
cat: 2
snake: 1
dog: 2
```

There're three fundamental concepts in Apache Beam, namely Pipeline, PCollection, and Transform.

* **Pipeline** holds the DAG (Directed Acyclic Graph) of data and process tasks. It's analogous to MapReduce `Job` and Storm `Topology`.
* **PCollection** is the data structure to which we apply various operations, like parse, convert, or aggregate. You can think of it as Spark `RDD`.
* And **Transform** is where your main logic goes. Each transform will take a PCollection in and produce a new PCollection. Beam provides many built-in Transforms, and we'll cover them later.

As in this example, `Pipeline` and `PipelineOptions` are used to construct a pipeline. Use the `with` statement so that context manager will invoke `Pipeline.run` and `wait_until_finish` automatically.

```
[Output PCollection] = [Input PCollection] | [Label] >> [Transform]
```

`|` is the operator to apply transforms, and each transform can be optionally supplied with a unique label. Transforms can be chained, and we can compose arbitrary shapes of transforms, and at runtime they'll be represented as DAG.

`beam.Create` is a transform that creates PCollection from memory data, mainly for testing. Beam has built-in sources and sinks to read and write bounded or unbounded data, and it's possible to implement our own.

`beam.Map` is a *one-to-one* transform, and in this example we convert a word string to a `(word, 1)` tuple. `beam.FlatMap` is a combination of `Map` and `Flatten`, i.e. we split each line into an array of words, and then flatten these sequences into a single one.

`CombinePerKey` works on two-element tuples. It groups the tuples by the first element (the key), and apply the provided function to the list of second elements (values). Finally, we use `beam.ParDo` to print out the counts. This is a rather basic transform, and we'll discuss it in the following section.

## Input and Output

Currently, Beam's Python SDK has very limited supports for IO. This table ([source][7]) gives an overview of the available built-in transforms:

| Language | File-based | Messaging | Database |
| --- | --- | --- | --- |
| Java | HDFS<br>TextIO<br>XML | AMQP<br>Kafka<br>JMS | Hive<br>Solr<br>JDBC |
| Python | textio<br>avroio<br>tfrecordio | - | Google Big Query<br>Google Cloud Datastore |

The following snippet demonstrates the usage of `textio`:

```python
lines = p | 'Read' >> beam.io.ReadFromText('/path/to/input-*.csv')
lines | 'Write' >> beam.io.WriteToText('/path/to/output', file_name_suffix='.csv')
```

`textio` is able to read multiple input files by using wildcard or you can flatten PCollections created from difference sources. The outputs are also split into several files due to pipeline's parallel processing nature.

## Transforms

There're basic transforms and higher-level built-ins. In general, we prefer to use the later so that we can focus on the application logic. The following table lists some commonly used higher-level transforms:

| Transform | Meaning |
| --- | --- |
| Create(value) | Creates a PCollection from an iterable. |
| Filter(fn) | Use callable `fn` to filter out elements. |
| Map(fn) | Use callable `fn` to do a one-to-one transformation. |
| FlatMap(fn) | Similar to `Map`, but `fn` needs to return an iterable of zero or more elements, and these iterables will be flattened into one PCollection. |
| Flatten() | Merge several PCollections into a single one. |
| Partition(fn) | Split a PCollection into several partitions. `fn` is a `PartitionFn` or a callable that accepts two arguments - `element`, `num_partitions`. |
| GroupByKey() | Works on a PCollection of key/value pairs (two-element tuples), groups by common key, and returns `(key, iter<value>)` pairs. |
| CoGroupByKey() | Groups results across several PCollections by key. e.g. input `(k, v)` and `(k, w)`, output `(k, (iter<v>, iter<w>))`. |
| RemoveDuplicates() | Get distint values in PCollection. |
| CombinePerKey(fn) | Similar to `GroupByKey`, but combines the values by a `CombineFn` or a callable that takes an iterable, such as `sum`, `max`. |
| CombineGlobally(fn) | Reduces a PCollection to a single value by applying `fn`. |

### Callable, DoFn, and ParDo

Most transforms accepts a callable as argument. In Python, [callable][8] can be a function, method, lambda expression, or class instance that has `__call__` method. Under the hood, Beam will wrap the callable as a `DoFn`, and all these transforms will invoke `ParDo`, the lower-level transform, with the `DoFn`.

Let's replace the expression `lambda x: x.split(' ')` with a `DoFn` class:

```python
class SplitFn(beam.DoFn):
    def process(self, element):
        return element.split(' ')

lines | beam.ParDo(SplitFn())
```

The `ParDo` transform works like `FlatMap`, except that it only accepts `DoFn`. In addition to `return`, we can `yield` element from `process` method:

```python
class SplitAndPairWithOneFn(beam.DoFn):
    def process(self, element):
        for word in element.split(' '):
            yield (word, 1)
```

### Combiner Functions

Combiner functions, or `CombineFn`, are used to reduce a collection of elements into a single value. You can either perform on the entire PCollection (`CombineGlobally`), or combine the values for each key (`CombinePerKey`). Beam is capable of wrapping callables into `CombinFn`. The callable should take an iterable and returns a single value. Since Beam distributes computation to multiple nodes, the combiner function will be invoked multiple times to get partial results, so they ought to be [commutative][9] and [associative][10]. `sum`, `min`, `max` are good examples.

Beam provides some built-in combiners like count, mean, top. Take count for instance, the following two lines are equivalent, they return the total count of lines.

```python
lines | beam.combiners.Count.Globally()
lines | beam.CombineGlobally(beam.combiners.CountCombineFn())
```

Other combiners can be found in Beam Python SDK Documentation ([link][12]). For more complex combiners, we need to subclass the `CombinFn` and implement four methods. Take the built-in `Mean` for an example:

[`apache_beam/transforms/combiners.py`][11]

```python
class MeanCombineFn(core.CombineFn):
  def create_accumulator(self):
    """Create a "local" accumulator to track sum and count."""
    return (0, 0)

  def add_input(self, (sum_, count), element):
    """Process the incoming value."""
    return sum_ + element, count + 1

  def merge_accumulators(self, accumulators):
    """Merge several accumulators into a single one."""
    sums, counts = zip(*accumulators)
    return sum(sums), sum(counts)

  def extract_output(self, (sum_, count)):
    """Compute the mean average."""
    if count == 0:
      return float('NaN')
    return sum_ / float(count)
```

### Composite Transform

Take a look at the [source code][13] of `beam.combiners.Count.Globally` we used before. It subclasses `PTransform` and applies some transforms to the PCollection. This forms a sub-graph of DAG, and we call it composite transform. Composite transforms are used to gather relative codes into logical modules, making them easy to understand and maintain.

```python
class Count(object):
  class Globally(ptransform.PTransform):
    def expand(self, pcoll):
      return pcoll | core.CombineGlobally(CountCombineFn())
```

More built-in transforms are listed below:

| Transform | Meaning |
| --- | --- |
| Count.Globally() ||
| Count.PerKey() ||
| Count.PerElement() ||
| Mean.Globally() ||
| Mean.PerKey() ||
| Top.Of(n, reverse) | Top.Largest(n), Top.Smallest(n) |
| Top.PerKey(n, reverse) | Top.LargestPerKey(n), Top.SmallestPerKey(n) |
| Sample.FixedSizeGlobally(n) ||
| Sample.FixedSizePerKey(n) ||
| ToList() ||
| ToDict() ||

## Windowing

## Pipeline Runner

https://beam.apache.org/documentation/runners/capability-matrix/

## References

* https://beam.apache.org/documentation/programming-guide/
* https://beam.apache.org/documentation/sdks/pydoc/2.1.0/
* https://sookocheff.com/post/dataflow/get-to-know-dataflow/

[1]: https://beam.apache.org/get-started/beam-overview/
[2]: https://web.archive.org/web/20160923141630/https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/35650.pdf
[3]: https://web.archive.org/web/20160201091359/http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/41378.pdf
[4]: https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101
[5]: https://github.com/pyenv/pyenv
[6]: https://www.python.org/downloads/source/
[7]: https://beam.apache.org/documentation/io/built-in/
[8]: https://docs.python.org/2/library/functions.html#callable
[9]: https://en.wikipedia.org/wiki/Commutative_property
[10]: https://en.wikipedia.org/wiki/Associative_property
[11]: https://github.com/apache/beam/blob/v2.1.0/sdks/python/apache_beam/transforms/combiners.py#L75
[12]: https://beam.apache.org/documentation/sdks/pydoc/2.1.0/apache_beam.transforms.html#module-apache_beam.transforms.combiners
[13]: https://github.com/apache/beam/blob/v2.1.0/sdks/python/apache_beam/transforms/combiners.py#L101
