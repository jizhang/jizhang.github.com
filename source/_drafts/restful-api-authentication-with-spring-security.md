---
title: RESTful API Authentication with Spring Security
categories: Programming
tags: [spring boot, spring security, restful]
---

When it comes to implementing user authentication in RESTful API server, there're several options like [Spring Security][1], [Apache Shiro][2], or writing our own version of Filters and Servlets. If the server already uses [Spring Boot][3], then Spring Security is really a good fit, for it integrates quite well with Spring Boot project, thanks to all those automatic configurations. However, Spring Security's login facility is originally built for web forms or basic HTTP authentication, while modern apps usually lean on RESTful API. We can either adapt the frontend client to utilizing the built-in login methods as in this tutorial [Spring Security and Angular JS][4], or write custom Filter to [extract user credentials from input JSON][5].

![Spring Security](images/spring-security.png)

Having said that, personally I still prefer to maintain a consistent API style in user authentication, and I don't want to write awkward logics with raw Servlet request/response objects instead of using what Spring MVC provides, i.e. `@RestController`, `@RequestBody`, form validation, etc. Luckily, Spring Security provides integration for Servlet API, so that we can login/logout user within the Controller. In this article, I will demonstrate how to use Spring Security to guard your RESTful API server, with the following functionalities:

* Login/logout with JSON API.
* Remember me.
* Return 401 for unauthenticated requests.
* Custom table for user data.
* CSRF protection.
* Session persistence.

<!-- more -->

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
