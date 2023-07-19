---
title: Store Custom Data in Spring MVC Request Context
tags:
  - spring
  - java
categories: Programming
date: 2022-07-05 08:04:41
---


When developing a web application with Spring MVC, you want to make some data available throughout the current request, like authentication information, request identifier, etc. These data are injected into a request-scoped context, and destroyed after the request ends. There are several ways to achieve that, and this article will demonstrate how.

## Use HttpServletRequest or WebRequest

Controller methods can delare an `HttpServletRequest` typed argument. When it is invoked, Spring will pass in an instance that contains information specific to the current request, like path and headers. It also provides a pair of methods that gets and sets custom attributes. For instance, Spring itself uses it to store application context, locale and theme resolver.

```java
@RestController
public class UserController {
  @GetMapping("/info")
  public String getInfo(HttpServletRequest request) {
    Object ctx = request.getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT");
    return String.valueOf(ctx);
  }
}
```

We can certainly use it to store our own data, like in a `Filter` that sets the user information.

```java
@Component
public class UserFilter extends OncePerRequestFilter {
  @Override
  protected void doFilterInternal(
      HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
      throws ServletException, IOException {

    request.setAttribute("user", new User("Jerry"));
    filterChain.doFilter(request, response);
  }
}
```

<!-- more -->

Spring also provides the `WebRequest` interface that abstracts away Java servlet class. Under the hood, they store data in the same place.

```java
@GetMapping("/current-user")
public String getCurrentUser(WebRequest request) {
  var user = (User) request.getAttribute("user", WebRequest.SCOPE_REQUEST);
  return user.getUsername();
}
```

`HttpServletRequest` can also be injected as a dependency. For example in a service class:

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class UserService {
  private final HttpServletRequest request;

  public User getFromRequest() {
    var user = (User) request.getAttribute("user");
    log.info("Get from HttpServletRequest: {}", user);
    return user;
  }
}
```

You may need some knowledge of [Project Lombok][1] to understand the code. In short, when Spring initializes this service bean, it passes in a **proxy** object of `HttpServletRequest`. When `getFromRequest` is invoked, the `request` variable within will point to the current servlet request instance.

As we can see, using `HttpServletRequest` is straightforward, but it has two disadvantages. First, it is not type safe, we need to cast the return value. Second, the service layer should not know of the HTTP request. The context information we pass to lower layers should be decoupled. These two problems can be solved by the next approach.

## Annotate context bean with @RequestScope

The default Spring [bean scope][2] is `singleton`, and there are other scopes like `prototype`, `request`, and `session`. When marked with `@RequestScope`, a new instance will be created for every HTTP request, and get destroyed accordingly.

```java
@Data
@Component
@RequestScope
public class CustomContext {
  private User user;
}
```

When injected as a dependency, Spring also wraps it with a proxy object.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class UserService {
  private final CustomContext context;

  public User getFromScoped() {
    log.info("Get from request-scoped context: {}", context.getUser());
    return context.getUser();
  }
}
```

Now the service has a typed context object, and it is not coupled with the HTTP layer.

## RequestContextHolder static method

There is a utility class `RequestContextHolder` from which we can get the `currentRequestAttributes`, latter is an implementation of `RequestAttributes` interface with `getAttribute` and `setAttribute` methods. The difference is this interface can be used to extract request-scoped attributes (stored in `HttpServletRequest`) *and* session-scoped attributes (in `HttpSession`). The `WebRequest` instance is actually backed by `RequestAttributes`, so is the `@RequestScope` annotation.

```java
public User getFromRequestContextHolder() {
  var user = (User) RequestContextHolder.currentRequestAttributes()
      .getAttribute("user", RequestAttributes.SCOPE_REQUEST);
  log.info("Get from RequestContextHolder: {}", user);
  return user;
}
```

Since `RequestContextHolder` is used via static methods, it is necessary to tackle the multithreading problems. The answer is obvious: `ThreadLocal`.

```java
public abstract class RequestContextHolder  {
  private static final ThreadLocal<RequestAttributes> requestAttributesHolder =
      new NamedThreadLocal<>("Request attributes");

  @Nullable
  public static RequestAttributes getRequestAttributes() {
    RequestAttributes attributes = requestAttributesHolder.get();
    return attributes;
  }
}
```

This gives us an idea of implementing the fourth approach, i.e. write our own thread-local request context.

## Thread-local request context

Each servlet request is handled in a separate thread, so we can use a thread-local object to hold the request-scoped context.

```java
@Component
public class CustomContextHolder {
  private static final ThreadLocal<CustomContext> holder = new ThreadLocal<>();

  public void set(CustomContext context) {
    holder.set(context);
  }

  public CustomContext get() {
    return holder.get();
  }

  public void remove() {
    holder.remove();
  }
}
```

Beware the thread that processes your request is borrowed from a thread pool, and you don't want your previous request info leaking into the next, so let's clean it up in the `Filter`.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class UserFilter extends OncePerRequestFilter {
  private final CustomContextHolder holder;

  @Override
  protected void doFilterInternal(
      HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
      throws ServletException, IOException {

    var user = new User("Jerry");
    var threadLocalContext = new CustomContext();
    threadLocalContext.setUser(user);
    holder.set(threadLocalContext);

    try {
      filterChain.doFilter(request, response);
    } finally {
      holder.remove();
      log.info("Remove custom context from thread local.");
    }
  }
}
```

Example code can be found on [GitHub][3].


[1]: https://projectlombok.org/
[2]: https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/core.html#beans-factory-scopes
[3]: https://github.com/jizhang/blog-demo/tree/master/request-context
