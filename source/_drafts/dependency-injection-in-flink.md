---
title: Dependency Injection in Flink
categories: [Big Data]
tags: [flink, guice, java]
---

## TL;DR

Compose dependency graph with [Guice][1]:

```java
public class DatabaseModule extends AbstractModule {
  @Provides @Singleton
  public DataSource provideDataSource() {
    return new HikariDataSource();
  }

  @Provides @Singleton
  public UserRepository provideUserRepository(DataSource dataSource) {
    return new UserRepositoryImpl(dataSource);
  }
}
```

Create singleton injector:

```java
public class AppInjector {
  private static class Holder {
    static final Injector INJECTOR = Guice.createInjector(new DatabaseModule());
  }

  public static void injectMembers(Object instance) {
    Holder.INJECTOR.injectMembers(instance);
  }
}
```

Use in Flink function:

```java
public class UserMapper extends RichMapFunction<Long, User> {
  @Inject
  transient UserRepository userRepository;

  @Override
  public void open(Configuration parameters) throws Exception {
    AppInjector.injectMembers(this);
  }

  @Override
  public User map(Long userId) throws Exception {
    Objects.requireNonNull(userId, "User ID is null");
    return userRepository.getById(userId).orElseThrow(() -> new RuntimeException("User not found"));
  }
}
```


## Motivation


<!-- more -->

* Motivation
    * Separation of concerns
    * Singleton, connection limit, share in slots
    * Testing
* Flink functions, serialization mechanism
    * Datastream, table api, sql
* Guice quick start
    * Define modules, compose modules
    * Provide, Named
    * Scope, Singleton
    * Implicit creation
* Datasource example
    * No DI, static member, lazy initialization
    * Serialize object, or config only
* Inject configurations, vs. ParameterTool
    * https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/datastream/application_parameters/
* Complex demo: properties, repository, service, guava cache, redis
* Testability, flink specifit testing
    * https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/datastream/testing/
* DI in flink source
    * Custom table source


## References
* https://github.com/google/guice/wiki/GettingStarted
* https://getindata.com/blog/writing-flink-jobs-using-spring-dependency-injection-framework/
* https://medium.com/airteldigital/designing-and-developing-a-real-time-streaming-platform-with-flink-and-google-guice-213b40e063de


[1]: https://github.com/google/guice
