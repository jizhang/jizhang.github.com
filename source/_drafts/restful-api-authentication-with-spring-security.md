---
title: RESTful API Authentication with Spring Security
categories: Programming
tags: [spring boot, spring security, restful]
---

When it comes to implementing user authentication in RESTful API server, there're several options like [Spring Security][1], [Apache Shiro][2], or writing our own version of Filters and Servlets. If the server already uses [Spring Boot][3], then Spring Security is really a good fit, for it integrates quite well with Spring Boot project, thanks to all those automatic configurations. However, Spring Security's login facility is originally built for web forms or basic HTTP authentication, while modern apps usually lean on RESTful API. We can either adapt the frontend client to utilizing the built-in login methods as in this tutorial [Spring Security and Angular JS][4], or write custom Filter to [extract user credentials from input JSON][5].

![Spring Security](images/spring-security.png)

Having said that, personally I still prefer to maintain a consistent API style in user authentication, and I don't want to write awkward logics with raw Servlet request/response in Filter, instead of using what Spring MVC provides, i.e. `@RestController`, `@RequestBody`, form validation, etc. Luckily, Spring Security provides integration for Servlet API, so that we can login/logout user within the Controller. In this article, I will demonstrate how to use Spring Security to guard your RESTful API server, with the following functions:

* Login/logout with JSON API.
* Return 401 for unauthenticated requests.
* Custom table for user data.
* Remember me.
* CSRF protection.
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

It's not that Spring Security doesn't come with good default to table schema, but we probably want to have more control over them or we already have a set of user tables. If you're interested, here's the link to the default [User Schema][6]. Instead, I'm using the following schema in this demo.

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

By default, Spring Security will guard all API endpoints including `/api/login`, so we first need to tell it back down at certain requests, by configuring the `SecurityFilterChain`:

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

## Retrieve user credentials and attempt logging in

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

Note the `User` class implements the `UserDetails` interface, which tells Spring Security that this class can be used for authentication. To wire it to the mechanism, we need another class that implements `UserDetailsService` interface, mainly for retrieving the `User` class from wherever we store them.

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

Form Servlet 3+, `HttpServletRequest` adds `login`/`logout` methods to help authenticate user credential programmatically, and Spring Security [integrates with this function][8]. So in our `/api/login` handler, we simply invoke this method:

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

* Functions
    * ~~Login with POST JSON~~
    * ~~Logout~~
    * ~~Remember me~~
    * ~~Session persistence~~
    * ~~User schema~~
    * RBAC
    * ~~Get current user~~
    * ~~401 redirect~~
    * ~~CSRF~~
    * JWT
    * Form validation
    * Error response
    * Other options
        * Built-in form login, http basic, jdbc dao
        * Custom filter

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
