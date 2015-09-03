---
layout: post
title: "View Spark Source in Eclipse"
date: 2015-09-01 18:38
comments: true
categories: [Notes, BigData]
published: false
---

Reading source code is a great way to learn opensource projects. I used to read Java projects' source code on [GrepCode](http://grepcode.com/) for it is online and has very nice cross reference features. As for Scala projects such as [Apache Spark](http://spark.apache.org), though its source code can be found on [GitHub](https://github.com/apache/spark/), it's quite necessary to setup an IDE to view the code more efficiently. Here's a howto of viewing Spark source code in Eclipse.

## Install Eclipse and Scala IDE Plugin

One can download Eclipse from [here](http://www.eclipse.org/downloads/). I recommend the "Eclipse IDE for Java EE Developers", which contains a lot of daily-used features.

![](/images/scala-ide.png)

Then go to Scala IDE's [official site](http://scala-ide.org/download/current.html) and install the plugin through update site or zip archive.

## Generate Project File with Maven

Spark is mainly built with Maven, so make sure you have Maven installed on your box, and download the latest Spark source code from [here](http://spark.apache.org/downloads.html), unarchive it, and execute the following command:

```bash
$ mvn -am -pl core -DskipTests package eclipse:eclipse
```

<!-- more -->

This command does a bunch of things. First, it indicates what modules should be built. Spark is a large project with multiple modules. Currently we're only interested in its core module, so `-pl` or `--projects` is used. `-am` or `--also-make` tells Maven to build core module's dependencies as well. We can see the module list in output:

```text
[INFO] Scanning for projects...
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Build Order:
[INFO]
[INFO] Spark Project Parent POM
[INFO] Spark Launcher Project
[INFO] Spark Project Networking
[INFO] Spark Project Shuffle Streaming Service
[INFO] Spark Project Unsafe
[INFO] Spark Project Core
```

`package` tells Maven to download all dependencies and compile the source code. If you encounter some `OutOfMemoryException`, try enlarging the heap size by setting `MAVEN_OPTS`:

```bash
$ MAVEN_OPTS=-Xmx1G mvn ...
```

`eclipse:eclipse` will generate the `.project` and `.classpath` files for Eclipse. But the result is not perfect, both files need some fixes.

Edit `core/.classpath`, change the following two lines:

```xml
<classpathentry kind="src" path="src/main/scala" including="**/*.java"/>
<classpathentry kind="src" path="src/test/scala" output="target/scala-2.10/test-classes" including="**/*.java"/>
```

to

```xml
<classpathentry kind="src" path="src/main/scala" including="**/*.java|**/*.scala"/>
<classpathentry kind="src" path="src/test/scala" output="target/scala-2.10/test-classes" including="**/*.java|**/*.scala"/>
```

Edit `core/.project`, make it looks like this:

```xml
<buildSpec>
  <buildCommand>
    <name>org.scala-ide.sdt.core.scalabuilder</name>
  </buildCommand>
</buildSpec>
<natures>
  <nature>org.scala-ide.sdt.core.scalanature</nature>
  <nature>org.eclipse.jdt.core.javanature</nature>
</natures>
```

Now you can import "Existing Projects into Workspace", including `core`, `launcher`, `network`, and `unsafe`.

## References

* http://docs.scala-lang.org/tutorials/scala-with-maven.html
* https://wiki.scala-lang.org/display/SIW/ScalaEclipseMaven
* https://cwiki.apache.org/confluence/display/SPARK/Useful+Developer+Tools
