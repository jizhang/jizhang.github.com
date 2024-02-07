---
title: Dependency Injection in Flink
tags:
  - flink
  - guice
  - java
categories:
  - Big Data
date: 2024-02-07 13:28:41
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

Dependency injection, or DI, is a common practice in Java programming, especially when you come from a Spring background. The most direct benefit is testability, meaning you can replace class implementation with test stub. Other benefits are separation of concerns, better class hierarchy, inversion of control, etc. Component defines its dependencies via class constructor or annotated members, and the DI framework creates a container, or context, to wire these components properly. This context is usually created at startup and lives through the application lifecycle. Some examples are Spring `ApplicationContext`, Guice `Injector`.

Flink is a distributed computing framework, and it is favorable to decouple business logic from it by dependency injection. However, Flink application is composed of functional classes, that are instantiated in driver class, or `main` method, serialized and sent to distributed task managers. We cannot inject dependencies into these classes unless all our components are serializable. Fortunately, Flink provides a lifecycle hook `open` that is called when the job starts. Combined with another common pattern, Singleton, we can make DI framework play well with Flink.

<!-- more -->


## Guice crash course

The dependency injection framework I choose is Guice, because it is simple, light-weight, and effective. Usually we declare class dependencies with constructor, add all components to a module, and let Guice do the rest.


### Declare dependencies

There are three ways to declare dependencies for a class. The constructor approach is preferable.

```java
import com.google.inject.Inject;
// Or import jakarta.inject.Inject;

// 1. Constructor
public class UserRepositoryImpl implements UserRepository {
  private DataSource dataSource;

  @Inject
  public UserRepositoryImpl(DataSource dataSource) {
    this.dataSource = dataSource;
  }
}

// 2. Member
class UserRepositoryImpl implements UserRepository {
  @Inject
  private DataSource dataSource;
}

// 3. Setter
public class UserRepositoryImpl implements UserRepository {
  private DataSource dataSource;

  @Inject
  public void setDataSource(DataSource dataSource) {
    this.dataSource = dataSource;
  }
}
```


### Add components to module

Module is a mechanism of Guice to configure the components. How to initialize them, which concrete class implements the interface, what to do when there are multiple implementations, etc. Components are grouped into modules, and modules can be grouped together themselves. There are plenty of topics here, one can refer to its [documentation][2], and I will cover some basic usage.

First, components can be created implicitly, as long as Guice can figure out the dependency graph solely by class type and annotation. For instance:

```java
@ImplementedBy(UserRepositoryImpl.class)
public interface UserRepository {}

public class UserRepositoryImpl implements UserRespository {
  @Inject
  private HikariDataSource dataSource;
}

var injector = Guice.createInjector();
injector.getInstance(UserRepository.class);
```

`dataSource` is typed `HikariDataSource`, which is a concrete class, so Guice knows how to create it. If it is `DataSource`, Guice would raise a missing implementation error. For `UserRepository`, however, Guice knows the implementation because we declare it by `ImplementedBy` annotation. Otherwise, we need to declare this relationship in a module:

```java
import com.google.inject.AbstractModule;
import com.google.inject.Provides;

// 1. Add bindings
public class DatabaseModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(UserRepository.class).to(UserRepositoryImpl.class);
  }
}

// 2. Use provider method
public class DatabaseModule extends AbstractModule {
  @Provides
  public UserRepository provideUserRepository(UserRepositoryImpl impl) {
    return impl;
  }
}

var injector = Guice.createInjector(new DatabaseModule());
injector.getInstance(UserRepository.class);
```

These two methods are equivalent. The second approach can be interpreted in this way:

* User requests for a `UserRepository` instance.
* Guice sees the `provideUserRepository` method, due to its annotation and return type.
* The method requires a `UserRepositoryImpl` parameter.
* Guice creates the implementation instance implicitly, because it is a concrete class.
* The method gets the instance, possibly modifies it, and returns it to the user.

The second approach is a little different from what we use before, where the parameter is `DataSource`, and we create `UserRepositoryImpl` manually:

```java
@Provides
public UserRepository provideUserRepository(DataSource dataSource) {
  return new UserRepositoryImpl(dataSource);
}
```

In this case, the `Inject` annotation in `UserRepositoryImpl` can be omitted, because Guice is not responsible for creating the instance, unless you deliberately try to get a `UserRepositoryImpl` instance from it.

In provider method, we can configure the instance we return:

```java
@Provides @Singleton
public DataSource provideDataSource() {
  var config = new HikariConfig();
  config.setJdbcUrl("jdbc:mysql://localhost:3306/flink_di");
  config.setUsername("root");
  config.setPassword("");
  return new HikariDataSource(config);
}
```

Lastly, modules can be grouped together:

```java
public class EtlModule extends AbstractModule {
  @Override
  protected void configure() {
    install(new ConfigModule());
    install(new DatabaseModule());
    install(new RedisModule());
  }
}

var injector = Guice.createInjector(new EtlModule());
```


### Named and scoped components

When there are multiple instances for a type with different configuration, use `Named` annotation to tell them apart. It is also possible to create [custom annotations][3], or use bindings in `AbstractModule#configure` instead of provider method.

```java
public class DatabaseModule extends AbstractModule {
  @Provides @Named("customer") @Singleton
  public DataSource provideCustomerDataSource() {
    return new HikariDataSource();
  }

  @Provides @Named("product") @Singleton
  public DataSource provideProductDataSource() {
    return new HikariDataSource();
  }
}

@Singleton
public class UserRepositoryImpl extends UserRepository {
  @Inject @Named("customer")
  private DataSource dataSource;
}
```

Both data sources and the implementation instance are annotated with `Singleton`, meaning Guice will return the same instance when it is requested. Otherwise, it works like the [prototype scope][4] in Spring.


## Flink pipeline serialization

Consider this simple pipeline that transforms a stream of ID to user models and print to the console.

```java
var env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStreamSource<Long> source = env.fromElements(1L);
DataStream<User> users = source.map(new UserMapper());
users.print();

env.execute();
```

Under the hood, Flink will build this pipeline into a job graph, serialize it, and send to remote task managers. The `map` operator takes a `MapFunction` implementation, in this case a `UserMapper` instance. This instance is wrapped in `SimpleUdfStreamOperatorFactory` and gets serialized with Java object serialization mechanism.

```java
// org.apache.flink.util.InstantiationUtil
public static byte[] serializeObject(Object o) throws IOException {
  try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
      ObjectOutputStream oos = new ObjectOutputStream(baos)) {
    oos.writeObject(o);
    oos.flush();
    return baos.toByteArray();
  }
}
```

Pipeline operators become a series of configuration hash maps and are sent to the job manager by a remote call.

```
org.apache.flink.configuration.Configuration {
  operatorName=Map,
  serializedUdfClassName=org.apache.flink.streaming.api.operators.SimpleUdfStreamOperatorFactory,
  serializedUDF=[B@6c67e137,
}
```

For `ObjectOutputStream` to work, every class in the pipeline must implement the `Serializable` interface, as well as their member fields. For `UserMapper`, it extends `RichMapFunction` which implements the `Serializable` interface. However, if we add a dependency and that object is not serializable, an error would occur:

```java
public class UserMapper extends RichMapFunction<Long, User> {
  @Inject
  UserRepository userRepository;
}

// main
var injector = Guice.createInjector(new DatabaseModule());
var userMapper = injector.getInstance(UserMapper.class);
DataStream<User> users = source.map(userMapper);
// java.io.NotSerializableException: com.zaxxer.hikari.pool.HikariPool$PoolEntryCreator
```

This is because `HikariDataSource` is not serializable. As a result, it is not possible to carry `userRepository` through serialization, but set it after `UserMapper` is restored and opened, as is demonstrated at the beginning of this article. We add `transient` keyword to inform Java to not include this field when serializing.

```java
public class UserMapper extends RichMapFunction<Long, User> {
  @Inject
  transient UserRepository userRepository;

  @Override
  public void open(Configuration parameters) throws Exception {
    AppInjector.injectMembers(this);
  }
}
```

In `AppInjector`, we use the Singleton pattern to ensure there is only one Guice injector, and Guice itself works in a thread-safe manner so heavy resources like connection pool can be shared among different user defined functions.


## Unit testing

As mentioned earlier, dependency injection improves testability. To test the `UserMapper`, we can mock the dependency and test it like a plain function. Other testing techniques can be found in the [documentation][5].

```java
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

public class UserMapperTest {
  @Test
  public void testMap() throws Exception {
    var userRepository = mock(UserRepository.class);
    when(userRepository.getById(1L))
        .thenReturn(Optional.of(new User(1L, "jizhang", new Date())));

    var userMapper = new UserMapper();
    userMapper.userRepository = userRepository;
    assertEquals("jizhang", userMapper.map(1L).getUsername());
  }
}
```


## References
* https://github.com/google/guice/wiki/GettingStarted
* https://getindata.com/blog/writing-flink-jobs-using-spring-dependency-injection-framework/
* https://medium.com/airteldigital/designing-and-developing-a-real-time-streaming-platform-with-flink-and-google-guice-213b40e063de


[1]: https://github.com/google/guice
[2]: https://github.com/google/guice/wiki/Bindings
[3]: https://github.com/google/guice/wiki/BindingAnnotations
[4]: https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html#beans-factory-scopes-prototype
[5]: https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/datastream/testing/
