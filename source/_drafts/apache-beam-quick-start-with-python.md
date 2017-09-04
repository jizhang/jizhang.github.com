---
title: Apache Beam Quick Start with Python
categories: [Big Data]
tags: [beam, python, mapreduce, stream processing]
---

<!-- more -->


```bash
brew install openssl
export CFLAGS="-I$(brew --prefix openssl)/include"
export LDFLAGS="-L$(brew --prefix openssl)/lib"
./configure --prefix=/usr/local/python-2.7
virtualenv venv --distribute
source venv/bin/activate
pip install apache-beam
```
