---
title: How to Avoid NullPointerException
tags:
  - java
  - spring
  - eclipse
categories: Programming
date: 2018-09-20 11:25:16
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

There are some coding conventions we can use to avoid `NullPointerException`.

* Use methods that already guard against `null` values, such as `String#equals`, `String#valueOf`, and third party libraries that help us check whether string or collection is empty.

```java
if (str != null && str.equals("text")) {}
if ("text".equals(str)) {}

if (obj != null) { obj.toString(); }
String.valueOf(obj); // "null"

// from spring-core
StringUtils.isEmpty(str);
CollectionUtils.isEmpty(col);
// from guava
Strings.isNullOrEmpty(str);
// from commons-collections4
CollectionUtils.isEmpty(col);
```

* If a method accepts nullable value, define two methods with different signatures, so as to make every parameter mandatory.

```java
public void methodA(Object arg1) {
  methodB(arg1, new Object[0]);
}

public void methodB(Object arg1, Object[] arg2) {
  for (Object obj : arg2) {} // no null check
}
```

* For return values, if the type is `Collection`, return an empty collection instead of null; if it's a single object, consider throwing an exception. This approach is also suggested by *Effective Java*. Good examples come from Spring's JdbcTemplate:

```java
// return new ArrayList<>() when result set is empty
jdbcTemplate.queryForList("SELECT 1");

// throws EmptyResultDataAccessException when record not found
jdbcTemplate.queryForObject("SELECT 1", Integer.class);

// works for generics
public <T> List<T> testReturnCollection() {
  return Collections.emptyList();
}
```

## Static Check

Java has some static code analysis tools, like Eclipse IDE, SpotBugs, Checker Framework, etc. They can find out program bugs during compilation process. It would be nice to catch `NullPointerException` as early as possible, and this can be done with annotations like `@Nullable` and `@Nonnull`.

However, nullness check annotations have not been standardized yet. Though there was a [JSR 305][4] proposed back to Sep. 2006, it has been dormant ever since. A lot of third party libraries provide such annotations, and they are supported by different tools. Some popular candidates are:

* `javax.annotation.Nonnull`, proposed by JSR 305, and its reference implementation is `com.google.code.findbugs.jsr305`.
* `org.eclipse.jdt.annotation.NonNull`, used by Eclipse IDE to do static nullness check.
* `edu.umd.cs.findbugs.annotations.NonNull`, used by SpotBugs, it depends on `jsr305`.
* `org.springframework.lang.NonNull`, provided by Spring Framework.
* `org.checkerframework.checker.nullness.qual.NonNull`, used by Checker Framework.
* `android.support.annotation.NonNull`, used by Android Development Toolkit.

I suggest using a cross IDE solution like SpotBugs or Checker Framework, which also plays nicely with Maven.

### `@NonNull` and `@CheckForNull` with SpotBugs

SpotBugs is the successor of FindBugs. We can use `@NonNull` and `@CheckForNull` on method arguments or return values, so as to apply nullness check. Notably, SpotBugs does not respect `@Nullable`, which is only useful when overriding `@ParametersAreNullableByDefault`. Use `@CheckForNull` instead.

To integrate SpotBugs with Maven and Eclipse, one can refer to its [official document][5]. Make sure you add the `spotbugs-annotations` package in Maven dependencies, which includes the nullness check annotations.

```xml
<dependency>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-annotations</artifactId>
    <version>3.1.7</version>
</dependency>
```

Here are the examples of different scenarios.

```java
@NonNull
private Object returnNonNull() {
  // ERROR: returnNonNull() may return null, but is declared @Nonnull
  return null;
}

@CheckForNull
private Object returnNullable() {
  return null;
}

public void testReturnNullable() {
  Object obj = returnNullable();
  // ERROR: Possible null pointer dereference due to return value of called method
  System.out.println(obj.toString());
}

private void argumentNonNull(@NonNull Object arg) {
  System.out.println(arg.toString());
}

public void testArgumentNonNull() {
  // ERROR: Null passed for non-null parameter of argumentNonNull(Object)
  argumentNonNull(null);
}

public void testNullableArgument(@CheckForNull Object arg) {
  // ERROR: arg must be non-null but is marked as nullable
  System.out.println(arg.toString());
}
```

For Eclipse users, it is also possible to use its built-in nullness check along with SpotBugs. By default, Eclipse uses annotations under its own package, i.e. `org.eclipse.jdt.annotation.Nullable`, but we can easily add more annotations.

![Eclipse null analysis](/images/java-npe/eclipse.png)

### `@NonNull` and `@Nullable` with Checker Framework

Checker Framework works as a plugin to the `javac` compiler, to provide type checks, detect and prevent various errors. Follow the [official document][6], integrate Checker Framework with `maven-compiler-plugin`, and it will start to work when executing `mvn compile`. The Nullness Checker supports all kinds of annotations, from JSR 305 to Eclipse built-ins, even `lombok.NonNull`.

```java
import org.checkerframework.checker.nullness.qual.Nullable;

@Nullable
private Object returnNullable() {
  return null;
}

public void testReturnNullable() {
  Object obj = returnNullable();
  // ERROR: dereference of possibly-null reference obj
  System.out.println(obj.toString());
}
```

By default, Checker Framework applies `@NonNull` to all method arguments and return values. The following snippet, without any annotations, cannot pass compilation, either.

```java
private Object returnNonNull() {
  // ERROR: incompatible types in return.
  // found: null, required: @Initialized @NonNull Object.
  return null;
}

private void argumentNonNull(Object arg) {
  System.out.println(arg.toString());
}

public void testArgumentNonNull() {
  // ERROR: incompatible types in argument.
  // found: null, required: @Initialized @NonNull Object
  argumentNonNull(null);
}
```

Checker Framework is especially useful for Spring Framework users, because from version 5.x, Spring provides built-in annotations for nullness check, and they are all over the framework code itself, mainly for Kotlin users, but we Java programmers can benefit from them, too. Take `StringUtils` class for instance, since the whole package is declared `@NonNull`, those methods with nullable argument and return values are explicitly annotated with `@Nullable`, so the following code will cause compilation failure.

```java
// Defined in spring-core
public abstract class StringUtils {
  // str inherits @NonNull from top-level package
  public static String capitalize(String str) {}

  @Nullable
  public static String getFilename(@Nullable String path) {}
}

// ERROR: incompatible types in argument. found null, required @NonNull
StringUtils.capitalize(null);

String filename = StringUtils.getFilename("/path/to/file");
// ERROR: dereference of possibly-null reference filename
System.out.println(filename.length());
```

## Optional Class

Java 8 introduces the `Optional<T>` class that can be used to wrap a nullable return value, instead of returning null or throwing an exception. On the upside, a method that returns `Optional` explicitly states it may return an empty value, so the invoker must check the presence of the value, and no NPE will be thrown. However, it does introduce more codes, and adds some overhead of object creation. So use it with caution.

```java
Optional<String> opt;

// create
opt = Optional.empty();
opt = Optional.of("text");
opt = Optional.ofNullable(null);

// test & get
if (opt.isPresent()) {
  opt.get();
}

// fall back
opt.orElse("default");
opt.orElseGet(() -> "default");
opt.orElseThrow(() -> new NullPointerException());

// operate
opt.ifPresent(value -> {
  System.out.println(value);
});
opt.filter(value -> value.length() > 5);
opt.map(value -> value.trim());
opt.flatMap(value -> {
  String trimmed = value.trim();
  return trimmed.isEmpty() ? Optional.empty() : Optional.of(trimmed);
});
```

Chaining of methods is a common cause of NPE, but if you have a series of methods that return `Optional`, you can chain them with `flatMap`, NPE-freely.

```java
String zipCode = getUser()
    .flatMap(User::getAddress)
    .flatMap(Address::getZipCode)
    .orElse("");
```

Java 8 [Stream API][7] also uses optionals to return nullable values. For instance:

```java
stringList.stream().findFirst().orElse("default");
stringList.stream()
    .max(Comparator.naturalOrder())
    .ifPresent(System.out::println);
```

Lastly, there are some special optional classes for primitive types, such as `OptionalInt`, `OptionalDouble`, etc. Use them whenever you find applicable.

## NPE in Other JVM Languages

Scala provides an [`Option`][8] class similar to Java 8 `Optional`. It has two subclasses, `Some` represents an existing value, and `None` for empty result.

```scala
val opt: Option[String] = Some("text")
opt.getOrElse("default")
```

Instead of invoking `Option#isEmpty`, we can use Scala's pattern match:

```scala
opt match {
  case Some(text) => println(text)
  case None => println("default")
}
```

Scala's collection operations are very powerful, and `Option` can be treated as collection, so we can apply `filter`, `map`, or for-comprehension to it.

```scala
opt.map(_.trim).filter(_.length > 0).map(_.toUpperCase).getOrElse("DEFAULT")
val upper = for {
  text <- opt
  trimmed <- Some(text.trim())
  upper <- Some(trimmed) if trimmed.length > 0
} yield upper
upper.getOrElse("DEFAULT")
```

Kotlin takes another approach. It distinguishes [nullable types and non-null types][9], and programmers are forced to check nullness before using nullable variables.

```kotlin
var a: String = "text"
a = null // Error: Null can not be a value of a non-null type String

val b: String? = "text"
// Error: Only safe (?.) or non-null asserted (!!.) calls are allowed
// on a nullable receiver of type String?
println(b.length)

val l: Int? = b?.length // safe call
b!!.length // may throw NPE
```

When calling Java methods from Kotlin, the compiler does not ensure null-safety, because every object from Java is nullable. But we can use annotations to achieve strict nullness check. Kotlin supports a wide range of [annotations][9], including those used in Spring Framework, which makes Spring API null-safe in Kotlin.

## Conclusion

In all these solutions, I prefer the annotation approach, since it's effective while less invasive. All public API methods should be annotated `@Nullable` or `@NonNull` so that the caller will be forced to do nullness check, making our program NPE free.

## References

* https://howtodoinjava.com/java/exception-handling/how-to-effectively-handle-nullpointerexception-in-java/
* http://jmri.sourceforge.net/help/en/html/doc/Technical/SpotBugs.shtml
* https://dzone.com/articles/features-to-avoid-null-reference-exceptions-java-a
* https://medium.com/@fatihcoskun/kotlin-nullable-types-vs-java-optional-988c50853692

[1]: https://docs.oracle.com/javase/7/docs/api/java/util/Objects.html
[2]: https://github.com/google/guava/wiki/PreconditionsExplained
[3]: https://projectlombok.org/features/NonNull
[4]: https://jcp.org/en/jsr/detail?id=305
[5]: https://spotbugs.readthedocs.io/en/latest/maven.html
[6]: https://checkerframework.org/manual/#maven
[7]: https://www.oracle.com/technetwork/articles/java/ma14-java-se-8-streams-2177646.html
[8]: https://www.scala-lang.org/api/current/scala/Option.html
[9]: https://kotlinlang.org/docs/reference/java-interop.html#nullability-annotations
