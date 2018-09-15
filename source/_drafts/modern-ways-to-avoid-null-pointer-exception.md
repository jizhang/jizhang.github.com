---
title: Modern Ways to Avoid NullPointerException
tags: [java]
categories: Programming
---

Java 空指针异常的若干解决方案

* old solutions
    * value == null
    * Strings#isBlank, Collections#isEmpty
    * return empty collections, not nulls. effective java #54
    * throw exception #55
    * "".equals, String.valueOf
    * argument is null? define two methods
    * state in comments
    * pay attention to
        * boxing
        * chaining calls
    * method argument
        * Objects.requireNonNull
        * guava preconditions https://github.com/google/guava/wiki/PreconditionsExplained
* optional #55
    * why? make the api clear
    * optional api
    * stream api
    * OptionalInt
* annotation + tools
    * checker framework https://checkerframework.org/manual/#nullness-checker
    * spring
        * https://www.baeldung.com/javax-validation-method-constraints
        * https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/lang/Nullable.html
    * eclipse
        * https://help.eclipse.org/photon/index.jsp?topic=%2Forg.eclipse.jdt.doc.user%2Ftasks%2Ftask-using_null_annotations.htm
        * https://sling.apache.org/documentation/development/null-analysis.html#use-with-eclipse
    * intellij https://www.jetbrains.com/help/idea/annotating-source-code.html
    * lombok
    * find bugs
        * https://stackoverflow.com/a/31157512/1030720
        * jsr305.jar
    * android https://medium.com/@jintin/nullable-nonnull-dba2fde14192
* npe in other jvm languages
    * scala
    * kotlin

<!-- more -->


## References

* https://howtodoinjava.com/java/exception-handling/how-to-effectively-handle-nullpointerexception-in-java/
* https://medium.com/@fatihcoskun/kotlin-nullable-types-vs-java-optional-988c50853692
* https://blogs.oracle.com/java-platform-group/java-8s-new-type-annotations

https://stackoverflow.com/questions/37598775/jsr-305-annotations-replacement-for-java-9
