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

There're basic transforms and higher-level built-ins. In general, we prefer to use the later so that we can focus on the application logic.

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

### CombinerFn

### DoFn

### PTransform

* combiners
  * Count
  * Mean
  * Sample
  * Top
  * ToDict
  * ToList

ParDo, DoFn, PTransform

## Windowing

## Pipeline Runner

https://beam.apache.org/documentation/runners/capability-matrix/

* install
  * python 2 with ssl (httpsconnection not found)
  * apache-beam package
  * wordcount direct mode
  * python only supports direct mode
* wordcount example
* concepts
  * Pipeline, pipeline runner
  * PCollection, bounded, unbounded
  * ParDo, DoFn, PTransform, General Requirements for Writing User Code for Beam Transforms
  * GroupByKey, CoGroupByKey (join)
  * Combine, CombinePerKey
  * Flatten, Partition
  * input / output
  * encoder, type hint
  * logging & testing https://beam.apache.org/get-started/wordcount-example/
* stream error log count
  * window, fixed, sliding
  * combinebykey
  * trigger ?
  * https://beam.apache.org/get-started/mobile-gaming-example/


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
