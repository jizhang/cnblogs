---
layout: post
title: "View Spark Source in Eclipse"
date: 2015-09-01 18:38
comments: true
categories: [Notes, Big Data]
published: true
---

Reading source code is a great way to learn opensource projects. I used to read Java projects' source code on [GrepCode](http://grepcode.com/) for it is online and has very nice cross reference features. As for Scala projects such as [Apache Spark](http://spark.apache.org), though its source code can be found on [GitHub](https://github.com/apache/spark/), it's quite necessary to setup an IDE to view the code more efficiently. Here's a howto of viewing Spark source code in Eclipse.

## Install Eclipse and Scala IDE Plugin

One can download Eclipse from [here](http://www.eclipse.org/downloads/). I recommend the "Eclipse IDE for Java EE Developers", which contains a lot of daily-used features.

![](/images/scala-ide.png)

Then go to Scala IDE's [official site](http://scala-ide.org/download/current.html) and install the plugin through update site or zip archive.

## Generate Project File with Maven

Spark is mainly built with Maven, so make sure you have Maven installed on your box, and download the latest Spark source code from [here](http://spark.apache.org/downloads.html), unarchive it, and execute the following command:

```bash
$ mvn -am -pl core dependency:resolve eclipse:eclipse
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

`dependency:resolve` tells Maven to download all dependencies. `eclipse:eclipse` will generate the `.project` and `.classpath` files for Eclipse. But the result is not perfect, both files need some fixes.

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

## Miscellaneous

### Access restriction: The type 'Unsafe' is not API

For module `spark-unsafe`, Eclipse will report an error "Access restriction: The type 'Unsafe' is not API (restriction on required library /path/to/jre/lib/rt.jar". To fix this, right click the "JRE System Library" entry in Package Explorer, change it to "Workspace default JRE".

### Download Sources and Javadocs

Add the following entry into pom's project / build / plugins:

```xml
<plugin>
    <artifactId>maven-eclipse-plugin</artifactId>
    <configuration>
        <downloadSources>true</downloadSources>
        <downloadJavadocs>true</downloadJavadocs>
    </configuration>
</plugin>
```

### build-helper-maven-plugin

Since Spark is a mixture of Java and Scala code, and the maven-eclipse-plugin only knows about Java source files, so we need to use build-helper-maven-plugin to include the Scala sources, as is described [here](http://docs.scala-lang.org/tutorials/scala-with-maven.html#integration-with-eclipse-scala-ide24). Fortunately, Spark's pom.xml has already included this setting.

## References

* http://docs.scala-lang.org/tutorials/scala-with-maven.html
* https://wiki.scala-lang.org/display/SIW/ScalaEclipseMaven
* https://cwiki.apache.org/confluence/display/SPARK/Useful+Developer+Tools
