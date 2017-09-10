---
title: Apache Beam Quick Start with Python
categories: [Big Data]
tags: [apache beam, python, mapreduce, stream processing]
---

[Apache Beam][1] is a big data processing standard created by Google in 2016. It provides unified DSL to process both batch and stream data, and can be executed on popular platforms like Spark, Flink, and of course Google's commercial product Dataflow. Beam's model is based on previous works known as [FlumeJava][2] and [Millwheel][3], and addresses solutions for data processing tasks like ETL, analysis, and [stream processing][4]. Currently it provides SDK in two languages, Java and Python. This article will introduce how to use Python to write Beam applications.

![Apache Beam Pipeline](/images/beam/arch.jpg)

## Installation

Apache Beam Python SDK requires Python 2.7.x. You can use [pyenv][5] to manage different Python versions, or compile from [source][6] (make sure you have SSL installed). And then you can install Beam SDK from PyPI, better in a virtual environment:

```bash
$ virtualenv venv --distribute
$ source venv/bin/activate
(venv) $ pip install apache-beam
````

<!-- more -->

## Wordcount Example

Wordcount is the de-facto "Hello World" in big data field, so let's take a look at how it's done with Beam:

```python
from __future__ import print_function
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
with beam.Pipeline(options=PipelineOptions()) as p:
    lines = p | 'Create' >> beam.Create(['cat dog', 'snake cat', 'fox dog'])
    counts = (
        lines
        | 'Split' >> (beam.FlatMap(lambda x: x.split(' '))
                      .with_output_types(unicode))
        | 'PairWithOne' >> beam.Map(lambda x: (x, 1))
        | 'GroupAndSum' >> beam.CombinePerKey(sum)
    )
    counts | 'Print' >> beam.ParDo(lambda (w, c): print('%s: %s' % (w, c)))
```

## Input and Output

## Transformation

## Windowing


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
