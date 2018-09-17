---
title: How to Avoid NullPointerException
tags: [java, spring, eclipse]
categories: Programming
---

`NullPointerException` happens when you dereference a possible `null` object without checking it. It's a common exception that every Java programmer may encounter in daily work. There're several strategies that can help us avoid this exception, making our codes more robust. In this article, I will list both traditional ways and those with tools and new features introduced by recent version of Java.

## Runtime Check

The most obvious way is to use `if (obj == null)` to check every variable you need to use, either from function argument, return value, or instance field. When you receive a `null` object, you can throw a different, more informative exception like `IllegalArgumentException`. There are some library functions that can make this process easier, like [`Objects#requireNonNull`][1]:

```java
public void testObjects(Object arg) {
  Object checked = Objects.requireNonNull(arg, "arg must not be null");
  checked.toString();
}
```

Or use Guava's [`Preconditions`][2] package, which provides all kinds of arguments checking facilities:

```java
public void testGuava(Object arg) {
  Object checked = Preconditions.checkNotNull(arg, "%s must not be null", "arg");
  checked.toString();
}
```

We can also let [Lombok][3] generate the check for us, which will throw a more meaningful `NullPointerException`:

```java
public void testLombok(@NonNull Object arg) {
  arg.toString();
}
```

The generated code and exception message are as follows:

```java
public void testLombokGenerated(Object arg) {
  if (arg == null) {
    throw new NullPointerException("arg is marked @NonNull but is null");
  }
  arg.toString();
}
```

This annotation can also be added to a class field, and Lombok will check nullness for every assignment.

<!-- more -->

## Coding Convention

* Strings#isBlank, Collections#isEmpty
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

## References

* https://howtodoinjava.com/java/exception-handling/how-to-effectively-handle-nullpointerexception-in-java/
* https://medium.com/@fatihcoskun/kotlin-nullable-types-vs-java-optional-988c50853692
* https://blogs.oracle.com/java-platform-group/java-8s-new-type-annotations


[1]: https://docs.oracle.com/javase/7/docs/api/java/util/Objects.html
[2]: https://github.com/google/guava/wiki/PreconditionsExplained
[3]: https://projectlombok.org/features/NonNull

https://stackoverflow.com/questions/37598775/jsr-305-annotations-replacement-for-java-9
https://javarevisited.blogspot.com/2013/05/ava-tips-and-best-practices-to-avoid-nullpointerexception-program-application.html
https://stackoverflow.com/questions/218384/what-is-a-nullpointerexception-and-how-do-i-fix-it


