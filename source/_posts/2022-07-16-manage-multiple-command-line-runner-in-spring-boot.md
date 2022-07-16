---
title: Manage Multiple CommandLineRunner in Spring Boot
tags:
  - spring
  - spring boot
  - java
categories: Programming
date: 2022-07-16 18:17:30
---


In Spring Boot, the [`CommandLineRunner`][1] and `ApplicationRunner` are two utility interfaces that we can use to execute code when application is started. However, all beans that implement these interfaces will be invoked by Spring Boot, and it takes some effort to execute only a portion of them. This is especially important when you are developing a console application with multiple entry points. In this article, we will use several techniques to achieve this goal.

## Put CommandLineRunner in different packages

By default, `@SpringBootApplication` will scan components (or beans) in current and descendant packages. When multiple `CommandLineRunner`s are discovered, Spring will execute them all. So the first approach will be separating those runners into different packages.

```java
package com.shzhangji.package_a;

@Slf4j
@SpringBootApplication
public class JobA implements CommandLineRunner {
  public static void main(String[] args) {
    SpringApplication.run(JobA.class, args);
  }

  @Override
  public void run(String... args) {
    log.info("Run package_a.JobA");
  }
}
```

If there is a `JobB` in `package_b`, these two jobs will not affect each other. But one problem is, when executing `JobA`, only components defined under `package_a` will be scanned. So if `JobA` wants to use a service in `com.shzhangji.common` package, we have to import this class explicitly:

```java
package com.shzhangji.package_a;

import com.shzhangji.common.UserService;

@SpringBootApplication
@Import(UserService.class)
public class JobA implements CommandLineRunner {
  @Autowired
  private UserService userService;
}
```

If there are multiple classes or packages that you want to import, you may as well change the base packages property:

```java
@SpringBootApplication(scanBasePackages = {
    "com.shzhangji.common",
    "com.shzhangji.package_a",
})
public class JobA implements CommandLineRunner {}
```

<!-- more -->

## Conditional scanning of components

So the basic idea is to expose only one `CommandLineRunner` to Spring's component scanning mechanism. Luckily Spring Framework provides the [`@Conditional`][2] annotation that can be used to filter beans based on system property, profile, or more complex conditions. As a matter of fact, Spring Boot's auto configuration feature is largely based on `@Conditional`. For instance:

```java
@ConditionalOnWebApplication
public class EmbeddedWebServerFactoryCustomizerAutoConfiguration {
  @ConditionalOnClass({ Tomcat.class, UpgradeProtocol.class })
  public static class TomcatWebServerFactoryCustomizerConfiguration {}

  @ConditionalOnClass({ Server.class, Loader.class, WebAppContext.class })
  public static class JettyWebServerFactoryCustomizerConfiguration {}
}
```

When initializing the embedded web server, Spring will check if Tomcat or Jetty is on the classpath (`@ConditionalOnClass`), and create the corresponding beans. The configuration class itself is also conditionally processed in a web environment (`@ConditionalOnWebApplication`).

In our situation, we shall use the [`@ConditionalOnProperty`][4] annotation, that filters beans based on system properties. Say we accept a property named `job`, and only create the `CommandLineRunner` bean when their values match.

```java
@Slf4j
@Component
@ConditionalOnProperty(name = "job", havingValue = "JobConditionalProperty")
public class JobConditionalProperty implements CommandLineRunner {
  @Override
  public void run(String... args) {
    log.info("Run JobConditionalProperty");
  }
}
```

To run this example in IDEA, use the following configuration:

![IDEA Config - ConditionalOnProperty](/images/command-line/conditional-on-property.png)

The `-Djob` in VM options and `--job` in program arguments are equivalent, so you only need to specify once. This property can also be set in a configuration file.

Similar to `@Conditional`, [Spring Profiles][5] can also be used to filter beans.

```java
@Slf4j
@Component
@Profile("JobByProfile")
public class JobByProfile implements CommandLineRunner {
  @Override
  public void run(String... args) throws Exception {
    log.info("Run JobByProfile");
  }
}
```

You can either activate the profile in command line arguments or environment variables.

![IDEA Config - Profile](/images/command-line/profile.png)

## Write a JobDispatcher

Lastly, we can always add a middle layer to solve the problem, i.e. a `JobDispatcher` that decides which `Runnable` to run. Only this time, we use the `ApplicationRunner` instead, because it will help us parsing the command line arguments.

```java
@Component
@RequiredArgsConstructor
public class JobDispatcher implements ApplicationRunner {
  private final AutowireCapableBeanFactory factory;

  @Override
  public void run(ApplicationArguments args) throws Exception {
    var jobArgs = args.getOptionValues("job");
    if (jobArgs == null || jobArgs.size() != 1) {
      throw new IllegalArgumentException("Invalid argument --job");
    }

    var jobClass = Class.forName("com.shzhangji.demo.commandline.dispatcher." + jobArgs.get(0));
    var job = (Runnable) factory.createBean(jobClass);
    job.run();
  }
}
```

When we pass `--job=JobDispatcherA` on the command line, the dispatcher will try to locate the job class, and initialize it with beans defined in context.

```java
@Slf4j
@RequiredArgsConstructor
public class JobDispatcherA implements Runnable {
  private final ApplicationContext context;

  @Override
  public void run() {
    log.info("Run JobDispatcherA in application context {}", context);
  }
}
```

Example code can be found on [GitHub][6].


[1]: https://docs.spring.io/spring-boot/docs/2.7.x/reference/htmlsingle/#features.spring-application.command-line-runner
[2]: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Conditional.html
[3]: https://docs.spring.io/spring-boot/docs/2.7.x/reference/htmlsingle/#features.developing-auto-configuration.understanding-auto-configured-beans
[4]: https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/condition/ConditionalOnProperty.html
[5]: https://docs.spring.io/spring-boot/docs/2.7.x/reference/htmlsingle/#features.profiles
[6]: https://github.com/jizhang/java-blog-demo/tree/master/command-line
