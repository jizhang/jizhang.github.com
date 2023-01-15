---
title: RESTful API Authentication with Spring Security
tags:
  - spring boot
  - spring security
  - restful
categories: Programming
date: 2023-01-15 14:51:51
---


When it comes to implementing user authentication in RESTful API server, there're several options like [Spring Security][1], [Apache Shiro][2], or writing our own version of Filters and Servlets. If the server already uses [Spring Boot][3], then Spring Security is really a good fit, for it integrates quite well with Spring Boot project, thanks to all those automatic configurations. However, Spring Security's login facility is originally built for web forms or basic HTTP authentication, while modern apps usually lean on RESTful API. We can either adapt the frontend client to utilizing the built-in login methods as in this tutorial [Spring Security and Angular JS][4], or write custom Filter to [extract user credentials from input JSON][5].

![Spring Security](images/spring-security.png)

Having said that, personally I still prefer to maintain a consistent API style in user authentication, and I don't want to write awkward logics with raw Servlet request/response objects in Filter, instead of using what Spring MVC provides, i.e. `@RestController`, `@RequestBody`, form validation, etc. Luckily, Spring Security provides integration for Servlet API, so that we can login/logout user within the Controller. In this article, I will demonstrate how to use Spring Security to guard your RESTful API server, with the following functions:

* Login/logout with JSON API.
* Return 401 for unauthenticated requests.
* Custom table for user data.
* CSRF protection.
* Remember me.
* Session persistence.

<!-- more -->

## Defining user authentication API

Let's define three APIs for user login, logout, and one that returns the currently logged-in user. All the requests and responses should be in the form of `application/json`.

```
POST /api/login
Request: {"username":"admin","password":"888888"}
Response: {"id":1,"nickname":"Jerry"}

POST /api/logout
Response: {}

GET /api/current-user
Response: {"id":1,"nickname":"Jerry"}
```

With Spring Boot, creating RESTful APIs is effortless. In the following example, we also add form validation and a custom exception handled by a global contoller. But these functions are beyond the scope of this article. The Spring Boot version I'm using is 3.x, with Spring Security 6.x, and Java 17.

```java
@RestController
@RequestMapping("/api")
public class AuthController {
  @PostMapping("/login")
  public CurrentUser login(@Valid @RequestBody LoginForm form, BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
      throw new AppException("Invalid username or password");
    }
    return new CurrentUser(1, "Jerry");
  }

  @PostMapping("/logout")
  public LogoutResponse logout() {
    return new LogoutResponse();
  }

  @GetMapping("/current-user")
  public CurrentUser getCurrentUser() {
    return new CurrentUser(1, "Jerry");
  }

  public record CurrentUser(Integer id, String nickname) {}
  public record LogoutResponse() {}
}
```

## Configure Spring Security filter chain

Add the Spring Security dependency into the project, along with the JDBC related ones, since we're going to retrieve user information from own version of `user` table. Note the dependency versions are managed by Spring Boot parent pom.

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
<dependency>
  <groupId>com.mysql</groupId>
  <artifactId>mysql-connector-j</artifactId>
  <scope>runtime</scope>
</dependency>
```

It's not that Spring Security doesn't come with good defaults for table schema, but we probably want to have more control over them or we already have a set of user tables. If you're interested, here's the link to the default [User Schema][6]. Instead, I'm using the following schema in this demo.

```sql
CREATE TABLE user (
  id INT NOT NULL PRIMARY KEY AUTO_INCREMENT
  ,username VARCHAR(255) NOT NULL
  ,password VARCHAR(255) NOT NULL
  ,nickname VARCHAR(255) NOT NULL
  ,created_at DATETIME NOT NULL
  ,updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
  ,UNIQUE KEY uk_username (username)
);

INSERT INTO user VALUES (1, 'admin', '{bcrypt}$2a$10$f4aQLof9kgM8mzJIP7a.Vuc3WYcQK8brcL6hrHdCdkzTH8AppEpOm', 'Jerry', NOW(), NOW());
```

The default password-hashing algorithm used by Spring Security is BCrypt. The following snippet can be used to generate such password digest. Other options can be found [here][7].

```java
var encoder = new BCryptPasswordEncoder();
var password = encoder.encode("888888");
System.out.println("{bcrypt}" + password);
```

By default, Spring Security will guard all API endpoints including `/api/login`, so we first need to tell it to back down at certain requests, by configuring the `SecurityFilterChain`:

```java
@SpringBootApplication
public class DemoApplication {
  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(customizer -> customizer
            .requestMatchers("/api/login").permitAll()
            .requestMatchers("/api/**").authenticated()
            .anyRequest().denyAll())
        .exceptionHandling(customizer -> customizer
            .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED)))
        .build();
  }
}
```

In addition, we tell Spring Security that when an unauthenticated user tries to access the restricted routes, it'll respond with 401 Unauthorized, so that the client, usually a single page application, can redirect to its login page. This facility is called authentication entry point. In the old days, it was the server's job to redirect to a login page, so the default entry point is an HTML page resided in the `/login` URL.

## Retrieve user credentials from database

Again, with Spring Boot, this task is much simplified. Let's create the `User` entity and its corresponding repository.

```java
@Table @Data
public class User implements UserDetails {
  private Integer id;
  private String username;
  private String password;
  private String nickname;
  private Date createdAt;
  private Date updatedAt;

  @Override
  public Collection<? extends GrantedAuthority> getAuthorities() { return Set.of(); }
  @Override
  public boolean isAccountNonExpired() { return true; }
  @Override
  public boolean isAccountNonLocked() { return true; }
  @Override
  public boolean isCredentialsNonExpired() { return true; }
  @Override
  public boolean isEnabled() { return true; }
}

@Repository
public interface UserRepository extends CrudRepository<User, Integer> {
  Optional<User> findByUsername(String username);
}
```

Note the `User` class implements the `UserDetails` interface, which tells Spring Security that this class can be used for authentication. To wire it into the mechanism, we need another class that implements `UserDetailsService` interface, mainly for retrieving the `User` instances from wherever we store them.

```java
@Service @RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {
  private final UserRepository repo;

  @Override
  public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    return repo.findByUsername(username)
        .orElseThrow(() -> new UsernameNotFoundException("Username " + username + " not found"));
  }
}
```

It'll find the table row by username, and use the aforementioned password encoder to check the authenticity.

## User login in Controller methods

From Servlet 3+, `HttpServletRequest` adds `login`/`logout` methods to help authenticate user credential programmatically, and Spring Security [integrates with this function][8]. So in our `/api/login` handler, we simply invoke this method:

```java
@PostMapping("/login")
public CurrentUser login(@Valid @RequestBody LoginForm form, BindingResult bindingResult,
                         HttpServletRequest request) {
  if (bindingResult.hasErrors()) {
    throw new AppException("Invalid username or password");
  }

  try {
    request.login(form.getUsername(), form.getPassword());
  } catch (ServletException e) {
    throw new AppException("Invalid username or password");
  }

  var auth = (Authentication) request.getUserPrincipal();
  var user = (User) auth.getPrincipal();
  log.info("User {} logged in.", user.getUsername());

  return new CurrentUser(user.getId(), user.getNickname());
}
```

`request.logout` can be used accordingly, and for `/api/current-user`, the `@AuthenticationPrincipal` annotation can be used on parameter to access the currently logged-in user:

```java
@GetMapping("/current-user")
public CurrentUser getCurrentUser(@AuthenticationPrincipal User user) {
  return new CurrentUser(user.getId(), user.getNickname());
}
```

Now we can test these APIs with [httpie][9], a commandline HTTP client:

```
% http localhost:8080/api/current-user
HTTP/1.1 401
Content-Length: 0
Date: Sun, 15 Jan 2023 04:10:51 GMT
```

As expected, since we're not logged in, the server responds with 401. Then let's try authenticate with username and password:

```
% http localhost:8080/api/login username=admin password=888888
HTTP/1.1 401
Content-Length: 0
Date: Sun, 15 Jan 2023 04:12:50 GMT
```

Unfortunately, the server denies us agian even if we provide the correct credential. The reason is Spring Security, by default, enables CSRF protection for all non-idempotent requests, such as POST, DELETE, etc. This can be disabled by configuration, and next section I'll show you how to use it properly to protect the API.

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
  return http
      .authorizeHttpRequests(customizer -> customizer)
      .csrf().disable()
      .build();
}
```

Now test the API again. Note that in the second request, we pass the Session ID as Cookie. You may notice the key `SESSION` is different from the default `JSESSIONID`, that is because I'm using Spring Session for session persistence, which I'll cover in the last section.

```
% http localhost:8080/api/login username=admin password=888888
HTTP/1.1 200
Content-Type: application/json
Date: Sun, 15 Jan 2023 04:20:45 GMT
Set-Cookie: SESSION=ZDZkOGQ5NTEtYmI4My00YjI2LTg3YzYtNDMzZTlkOWRmZDYz; Path=/; HttpOnly; SameSite=Lax
{
    "id": 1,
    "nickname": "Jerry"
}

% http localhost:8080/api/current-user Cookie:SESSION=ZDZkOGQ5NTEtYmI4My00YjI2LTg3YzYtNDMzZTlkOWRmZDYz
HTTP/1.1 200
Content-Type: application/json
Date: Sun, 15 Jan 2023 04:21:03 GMT
{
    "id": 1,
    "nickname": "Jerry"
}
```

## Enable CSRF protection

CSRF protection prevents malicious site from tricking user to submit a form unwillingly. Every form will be embedded with a server-generated token known as the CSRF token. Since the token cannot be attained by third-party, and it is validated in every submission, thus making the request safe. In the old days, again, web forms are generated on server side, while the token is saved in a hidden `<input>` and got submitted together with the form data. For instance, in Thymeleaf the token can be retrieved by a request attribute named `_csrf`:

```html
<input
  type="hidden"
  th:name="${_csrf.parameterName}"
  th:value="${_csrf.token}" />
```

But with SPA (Single Page Application), we need another way to retrieve the token. One approach is mentioned in the Angular tutorial I linked to earlier, in which the CSRF token is saved in Cookie, and every Ajax POST request is equipped with a header containing this token. Here I take a different approach, that is creating a dedicated endpoint for token retrieval:

```java
@GetMapping("/csrf")
public CsrfResponse csrf(HttpServletRequest request) {
  var csrf = (CsrfToken) request.getAttribute("_csrf");
  return new CsrfResponse(csrf.getToken());
}

public record CsrfResponse(String token) {}
```

This API should also be excluded from Spring Security:

```java
requestMatchers("/api/csrf").permitAll()
```

The client could fetch the CSRF token when it needs to do a POST/DELETE request. This token can also be cached in `localStorage` for further use, as long as the session is not timed out. Don't forget to clear the cache when user logs out.

```js
async function getCsrfToken() {
  const response = await fetch('/api/csrf')
  const payload = await response.json()
  return payload.token
}

async function login(username, password) {
  const response = await fetch('/api/login', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-CSRF-TOKEN': await getCsrfToken(),
    },
    body: JSON.stringify({ username, password }),
  })
  return await response.json()
}
```

## Remember-me authentication

When implementing this demo, the most tricky part is to utilize Spring Security's built-in remember-me authentication, in that Spring Security basically functions as a series of Filters, so when I decide to authenticate user in Controller instead of Filter, there'll be some extra work to do. Normally, with form login or filter-based auth, remember-me can be switched on by the following config:

```java
http.rememberMe(customizer -> customizer.alwaysRemember(true).key("demo"))
```

Under the hood, when user has logged in successfully, `RememberMeServices#loginSuccess` is invoked to generate and save a `remember-me` Cookie to the client. Next time the user can login without providing username and password.

```
% http localhost:8080/api/login username=admin password=888888 \
    Cookie:SESSION=YTI3ODMzZDctMjJlOC00MzNhLWIxYjItMTJkYzlhZDE2ZmM3 \
    X-CSRF-TOKEN:7NABU1UXxYeZH3GQf0G4NB0qGEiZwc0yIPR95Cte7jBWnYDc2-EzZTRzpuG0e0eoSWyMVi4YNSmo96wfQ8NE3Bg92QZhq7Pt

HTTP/1.1 200
Content-Type: application/json
Date: Sun, 15 Jan 2023 05:59:41 GMT
Set-Cookie: remember-me=YWRtaW46MTY3NDk3MTk4MTAwNDpTSEEyNTY6YmY3NjAwMmU0ODg3ZTFiMzgxMDBhNWEyMzM1NDgxOWYzODgwYmIxM2JlMzhmNjM2MjA1MGM0MWNkMjA1YWY1Yg; Max-Age=1209600; Expires=Sun, 29 Jan 2023 05:59:41 GMT; Path=/; HttpOnly
{
    "id": 1,
    "nickname": "Jerry"
}


% http localhost:8080/api/current-user \
    Cookie:remember-me=YWRtaW46MTY3NDk3MTk4MTAwNDpTSEEyNTY6YmY3NjAwMmU0ODg3ZTFiMzgxMDBhNWEyMzM1NDgxOWYzODgwYmIxM2JlMzhmNjM2MjA1MGM0MWNkMjA1YWY1Yg

HTTP/1.1 200
Content-Type: application/json
Date: Sun, 15 Jan 2023 05:59:59 GMT
Set-Cookie: SESSION=NDA4NjEwM2ItNTY2YS00ZDFlLWFiNjEtOTJjNGI2MGE4MTlj; Path=/; HttpOnly; SameSite=Lax
{
    "id": 1,
    "nickname": "Jerry"
}
```

Unfortunately, `HttpServletRequest#login` does not call `RememberMeServices#loginSuccess` for us, so we need to invoke the method by ourselves. Worse still, the `RememberMeServices` instance, in this case `TokenBasedRememberMeServices`, is only available within the Filter chain, meaning it is not registered in the Spring IoC container. After some digging in the source code, I managed to expose this instance to other Spring components.

```java
@SpringBootApplication
public class DemoApplication {
  @Autowired
  private ConfigurableBeanFactory beanFactory;

  @Bean("securityFilterChain")
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    var chain = http
        .authorizeHttpRequests(customizer -> customizer)
        .rememberMe(customizer -> customizer.alwaysRemember(true).key("demo"))
        .build();

    var rememberMeServices = http.getSharedObject(RememberMeServices.class);
    beanFactory.registerSingleton("rememberMeServices", rememberMeServices);

    return chain;
  }
}

@RestController
@RequiredArgsConstructor
@DependsOn("securityFilterChain")
public class AuthController {
  private final RememberMeServices rememberMeServices;
}
```

A `RememberMeServices` instance is created in the configuration phase by Spring Security, and we save it into the IoC container, making it available in the `AuthController`. The `@DependsOn` annotation ensures that `RememberMeServices` is registered before the `AuthController` is created. Next, the `loginSuccess` method can be invoked like this:

```java
@PostMapping("/login")
public CurrentUser login(@Valid @RequestBody LoginForm form, BindingResult bindingResult,
                         HttpServletRequest request, HttpServletResponse response) {
  request.login(form.getUsername(), form.getPassword());
  var auth = (Authentication) request.getUserPrincipal();
  var user = (User) auth.getPrincipal();
  rememberMeServices.loginSuccess(request, response, auth);
  return new CurrentUser(user.getId(), user.getNickname());
}
```

## Session persistence

Login state and CSRF token are stored in HTTP Session, and by default Session data are kept in Java process memory, so when the server restarts or there're multiple backends, users may need to login several times. The solution is simple, use Spring Session to store data in a third-party persistent storage. Take Redis for an example.

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.session</groupId>
  <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

Due to Spring Boot's auto-configuration feature, adding the dependencies will suffice to use Redis as the Session storage. To specify the Redis instance in production, add the following configs in `application.properties`.

```
spring.redis.host=localhost
spring.redis.port=6379
```

The demo project can be found on [GitHub][10].

## References
* https://docs.spring.io/spring-security/reference/index.html
* https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.security
* https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-web-security


[1]: https://spring.io/projects/spring-security
[2]: https://shiro.apache.org/
[3]: https://spring.io/projects/spring-boot
[4]: https://spring.io/guides/tutorials/spring-security-and-angular-js/
[5]: https://ckinan.com/blog/spring-security-credentials-from-json-request/
[6]: https://docs.spring.io/spring-security/reference/servlet/appendix/database-schema.html#_user_schema
[7]: https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html
[8]: https://docs.spring.io/spring-security/reference/servlet/integrations/servlet-api.html#servletapi-3
[9]: https://httpie.io/
[10]: https://github.com/jizhang/java-blog-demo/tree/master/api-auth
