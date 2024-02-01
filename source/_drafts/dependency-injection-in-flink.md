---
title: Dependency Injection in Flink
categories: [Big Data]
tags: [flink, guice, java]
---

## TL;DR

* Injector singleton
* Use in open

<!-- more -->

* Motivation
    * Separation of concerns
    * Singleton, connection limit, share in slots
    * Testing
* Flink functions, serialization mechanism
* Guice quick start
    * Define modules, compose modules
    * Provide, Named
    * Scope, Singleton
* Datasource example
    * No DI, static member, lazy initialization
    * Serialize object, or config only
* Inject configurations, vs. ParameterTool
* Complex demo: properties, repository, service, guava cache, redis
* Testability, flink specifit testing
* DI in flink source


## References
* https://github.com/google/guice/wiki/GettingStarted
* https://getindata.com/blog/writing-flink-jobs-using-spring-dependency-injection-framework/
* https://medium.com/airteldigital/designing-and-developing-a-real-time-streaming-platform-with-flink-and-google-guice-213b40e063de
