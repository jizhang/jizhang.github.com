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

  private AppInjector() {}

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

Dependency injection, or DI, is a common practice in Java programming, especially when you have a Spring background. The most direct benefit is testability, meaning you can replace class implementation with test stub. Other benefits are separation of concerns, better class hierarchy, inversion of control, etc. Component defines its dependencies via class constructor or annotated members, and the DI framework creates a container, or context, to wire these components properly. This context is usually created at startup and lives through the application lifecycle. Some examples are Spring `ApplicationContext`, Guice `Injector`.

Flink is a distributed computing framework, and it is favorable to decouple business logic from it by dependency injection. However, Flink application is composed of functional classes, that are instantiated in driver class, or `main` method, serialized and sent to distributed task managers. We cannot inject dependencies into these classes unless all our components are serializable. Fortunately, Flink provides a lifecycle hook `open` that is called when the job starts. Combined with another common pattern, Singleton, we can make DI framework play well with Flink.

<!-- more -->


## Guice crash course



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
