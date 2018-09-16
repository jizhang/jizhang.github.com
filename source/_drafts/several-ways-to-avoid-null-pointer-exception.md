---
title: Several Ways to Avoid NullPointerException
tags: [java, spring, eclipse]
categories: Programming
---




## Runtime Check

* value == null
* third party library
    * Strings#isBlank, Collections#isEmpty
    * method argument
        * Objects.requireNonNull
        * guava preconditions https://github.com/google/guava/wiki/PreconditionsExplained
    * lombok

## Coding Convention

* return empty collections, not nulls. effective java #54
* throw exception #55
* "".equals, String.valueOf
* argument is null? define two methods
* pay attention to
    * boxing
    * chaining calls
* null object pattern

## Static Check

different annotations
spotbugs
eclipse - config & spotbugs

* annotation + tools
    * checker framework
        * https://checkerframework.org/manual/#nullness-checker
        * https://checkerframework.org/manual/#maven
    * spring
        * https://www.baeldung.com/javax-validation-method-constraints
        * https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/lang/Nullable.html
    * eclipse
        * https://help.eclipse.org/photon/index.jsp?topic=%2Forg.eclipse.jdt.doc.user%2Ftasks%2Ftask-using_null_annotations.htm
        * https://sling.apache.org/documentation/development/null-analysis.html#use-with-eclipse
    * intellij https://www.jetbrains.com/help/idea/annotating-source-code.html
    * find bugs
        * https://stackoverflow.com/a/31157512/1030720
        * jsr305.jar
    * android https://medium.com/@jintin/nullable-nonnull-dba2fde14192
    * java 8 type annotation https://www.oracle.com/technetwork/articles/java/ma14-architect-annotations-2177655.html
    * various annotations https://stackoverflow.com/questions/4963300/which-notnull-java-annotation-should-i-use

## Optional Class

* optional #55
    * why? make the api clear
    * optional api
    * stream api
    * OptionalInt
https://dzone.com/articles/features-to-avoid-null-reference-exceptions-java-a

## NPE in Other Languages

Scala `Option`, `Some`, `None`
Typed Clojure union
    https://frenchy64.github.io/2013/10/04/null-pointer.html
    https://github.com/clojure-cookbook/clojure-cookbook/blob/master/10_testing/10-07_avoid-null-pointer.asciidoc
Kotlin nullable types
    * call java from kotlin, spring

<!-- more -->


## References

* https://howtodoinjava.com/java/exception-handling/how-to-effectively-handle-nullpointerexception-in-java/
* https://medium.com/@fatihcoskun/kotlin-nullable-types-vs-java-optional-988c50853692
* https://blogs.oracle.com/java-platform-group/java-8s-new-type-annotations

https://stackoverflow.com/questions/37598775/jsr-305-annotations-replacement-for-java-9
https://javarevisited.blogspot.com/2013/05/ava-tips-and-best-practices-to-avoid-nullpointerexception-program-application.html
https://stackoverflow.com/questions/218384/what-is-a-nullpointerexception-and-how-do-i-fix-it
