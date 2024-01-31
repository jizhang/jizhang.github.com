---
title: Dependency Injection in Flink
categories: [Big Data]
tags: [flink, guice, java]
---

TL;DR

* Injector singleton (enum)
* Use in open

<!-- more -->

* Flink functions, serialzation mechanism
* Guice quick start
* Datasource example
    * No DI, static member, lazy initialization
    * Serialize object, or config only
* Inject configurations, vs. ParameterTool
* DI in flink source


## References
* https://github.com/google/guice/wiki/GettingStarted
* https://getindata.com/blog/writing-flink-jobs-using-spring-dependency-injection-framework/
* https://medium.com/airteldigital/designing-and-developing-a-real-time-streaming-platform-with-flink-and-google-guice-213b40e063de
