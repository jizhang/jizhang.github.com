---
title: Apache Beam Quick Start with Python
categories: [Big Data]
tags: [apache beam, python, mapreduce, stream processing]
---

![Apache Beam Pipeline](/images/beam-pipeline.png)

<!-- more -->

* install
  * python 2 with ssl (httpsconnection not found)
  * apache-beam package
  * wordcount direct mode
  * python only supports direct mode
* wordcount example
* concepts
  * Pipeline, pipeline runner
  * PCollection
  * ParDo, DoFn, PTransform
  * input / output
  * logging & testing https://beam.apache.org/get-started/wordcount-example/
* stream error log count
  * unbounded pipeline
  * https://beam.apache.org/get-started/mobile-gaming-example/
* dataflow paper
* difference from spark

## References

* https://beam.apache.org/documentation/programming-guide/


```bash
brew install openssl
export CFLAGS="-I$(brew --prefix openssl)/include"
export LDFLAGS="-L$(brew --prefix openssl)/lib"
./configure --prefix=/usr/local/python-2.7
virtualenv venv --distribute
source venv/bin/activate
pip install apache-beam
```
