---
title: Store Custom Data in Spring MVC Request Context
tags: [spring, java]
categories: Programming
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


[1]: https://projectlombok.org/
[2]: https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/core.html#beans-factory-scopes
