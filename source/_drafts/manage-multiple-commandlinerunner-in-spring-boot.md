---
title: Manage Multiple CommandLineRunner in Spring Boot
categories: Programming
tags: [spring, spring boot, java]
---

In Spring Boot, the [`CommandLineRunner`][1] and `ApplicationRunner` are two utility interfaces that we can use to execute code when application is started. However, all beans that implement these interfaces will be invoked by Spring Boot, and it takes some effort to execute only a portion of them. This is especially important when you are developing a console application with Spring Boot. In this article, we will use several techniques to achieve this goal.

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

[1]: https://docs.spring.io/spring-boot/docs/2.7.x/reference/htmlsingle/#features.spring-application.command-line-runner
