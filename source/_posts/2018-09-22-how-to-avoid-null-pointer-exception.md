---
title: Java 空指针异常的若干解决方案
tags:
  - java
  - spring
  - eclipse
categories: Programming
date: 2018-09-22 20:27:36
---


Java 中任何对象都有可能为空，当我们调用空对象的方法时就会抛出 `NullPointerException` 空指针异常，这是一种非常常见的错误类型。我们可以使用若干种方法来避免产生这类异常，使得我们的代码更为健壮。本文将列举这些解决方案，包括传统的空值检测、编程规范、以及使用现代 Java 语言引入的各类工具来作为辅助。

## 运行时检测

最显而易见的方法就是使用 `if (obj == null)` 来对所有需要用到的对象来进行检测，包括函数参数、返回值、以及类实例的成员变量。当你检测到 `null` 值时，可以选择抛出更具针对性的异常类型，如 `IllegalArgumentException`，并添加消息内容。我们可以使用一些库函数来简化代码，如 Java 7 开始提供的 [`Objects#requireNonNull`][1] 方法：

```java
public void testObjects(Object arg) {
  Object checked = Objects.requireNonNull(arg, "arg must not be null");
  checked.toString();
}
```

Guava 的 [`Preconditions`][2] 类中也提供了一系列用于检测参数合法性的工具函数，其中就包含空值检测：

```java
public void testGuava(Object arg) {
  Object checked = Preconditions.checkNotNull(arg, "%s must not be null", "arg");
  checked.toString();
}
```

我们还可以使用 [Lombok][3] 来生成空值检测代码，并抛出带有提示信息的空指针异常：

```java
public void testLombok(@NonNull Object arg) {
  arg.toString();
}
```

生成的代码如下：

```java
public void testLombokGenerated(Object arg) {
  if (arg == null) {
    throw new NullPointerException("arg is marked @NonNull but is null");
  }
  arg.toString();
}
```

这个注解还可以用在类实例的成员变量上，所有的赋值操作会自动进行空值检测。

<!-- more -->

## 编程规范

通过遵守某些编程规范，也可以从一定程度上减少空指针异常的发生。

* 使用那些已经对 `null` 值做过判断的方法，如 `String#equals`、`String#valueOf`、以及三方库中用来判断字符串和集合是否为空的函数：

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

* 如果函数的某个参数可以接收 `null` 值，考虑改写成两个函数，使用不同的函数签名，这样就可以强制要求每个参数都不为空了：

```java
public void methodA(Object arg1) {
  methodB(arg1, new Object[0]);
}

public void methodB(Object arg1, Object[] arg2) {
  for (Object obj : arg2) {} // no null check
}
```

* 如果函数的返回值是集合类型，当结果为空时，不要返回 `null` 值，而是返回一个空的集合；如果返回值类型是对象，则可以选择抛出异常。Spring JdbcTemplate 正是使用了这种处理方式：

```java
// 当查询结果为空时，返回 new ArrayList<>()
jdbcTemplate.queryForList("SELECT * FROM person");

// 若找不到该条记录，则抛出 EmptyResultDataAccessException
jdbcTemplate.queryForObject("SELECT age FROM person WHERE id = 1", Integer.class);

// 支持泛型集合
public <T> List<T> testReturnCollection() {
  return Collections.emptyList();
}
```

## 静态代码分析

Java 语言有许多静态代码分析工具，如 Eclipse IDE、SpotBugs、Checker Framework 等，它们可以帮助程序员检测出编译期的错误。结合 `@Nullable` 和 `@Nonnull` 等注解，我们就可以在程序运行之前发现可能抛出空指针异常的代码。

但是，空值检测注解还没有得到标准化。虽然 2006 年 9 月社区提出了 [JSR 305][4] 规范，但它长期处于搁置状态。很多第三方库提供了类似的注解，且得到了不同工具的支持，其中使用较多的有：

* `javax.annotation.Nonnull`：由 JSR 305 提出，其参考实现为 `com.google.code.findbugs.jsr305`；
* `org.eclipse.jdt.annotation.NonNull`：Eclipse IDE 原生支持的空值检测注解；
* `edu.umd.cs.findbugs.annotations.NonNull`：SpotBugs 使用的注解，基于 `findbugs.jsr305`；
* `org.springframework.lang.NonNull`：Spring Framework 5.0 开始提供；
* `org.checkerframework.checker.nullness.qual.NonNull`：Checker Framework 使用；
* `android.support.annotation.NonNull`：集成在安卓开发工具中；

我建议使用一种跨 IDE 的解决方案，如 SpotBugs 或 Checker Framework，它们都能和 Maven 结合得很好。

### SpotBugs 与 `@NonNull`、`@CheckForNull`

SpotBugs 是 FindBugs 的后继者。通过在方法的参数和返回值上添加 `@NonNull` 和 `@CheckForNull` 注解，SpotBugs 可以帮助我们进行编译期的空值检测。需要注意的是，SpotBugs 不支持 `@Nullable` 注解，必须用 `@CheckForNull` 代替。如官方文档中所说，仅当需要覆盖 `@ParametersAreNonnullByDefault` 时才会用到 `@Nullable`。

[官方文档][5] 中说明了如何将 SpotBugs 应用到 Maven 和 Eclipse 中去。我们还需要将 `spotbugs-annotations` 加入到项目依赖中，以便使用对应的注解。

```xml
<dependency>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-annotations</artifactId>
    <version>3.1.7</version>
</dependency>
```

以下是对不同使用场景的说明：

```java
@NonNull
private Object returnNonNull() {
  // 错误：returnNonNull() 可能返回空值，但其已声明为 @Nonnull
  return null;
}

@CheckForNull
private Object returnNullable() {
  return null;
}

public void testReturnNullable() {
  Object obj = returnNullable();
  // 错误：方法的返回值可能为空
  System.out.println(obj.toString());
}

private void argumentNonNull(@NonNull Object arg) {
  System.out.println(arg.toString());
}

public void testArgumentNonNull() {
  // 错误：不能将 null 传递给非空参数
  argumentNonNull(null);
}

public void testNullableArgument(@CheckForNull Object arg) {
  // 错误：参数可能为空
  System.out.println(arg.toString());
}
```

对于 Eclipse 用户，还可以使用 IDE 内置的空值检测工具，只需将默认的注解 `org.eclipse.jdt.annotation.Nullable` 替换为 SpotBugs 的注解即可：

![Eclipse null analysis](/images/java-npe/eclipse.png)

### Checker Framework 与 `@NonNull`、`@Nullable`

Checker Framework 能够作为 `javac` 编译器的插件运行，对代码中的数据类型进行检测，预防各类问题。我们可以参照 [官方文档][6]，将 Checker Framework 与 `maven-compiler-plugin` 结合，之后每次执行 `mvn compile` 时就会进行检查。Checker Framework 的空值检测程序支持几乎所有的注解，包括 JSR 305、Eclipse、甚至 `lombok.NonNull`。

```java
import org.checkerframework.checker.nullness.qual.Nullable;

@Nullable
private Object returnNullable() {
  return null;
}

public void testReturnNullable() {
  Object obj = returnNullable();
  // 错误：obj 可能为空
  System.out.println(obj.toString());
}
```

Checker Framework 默认会将 `@NonNull` 应用到所有的函数参数和返回值上，因此，即使不添加这个注解，以下程序也是无法编译通过的：

```java
private Object returnNonNull() {
  // 错误：方法声明为 @NonNull，但返回的是 null。
  return null;
}

private void argumentNonNull(Object arg) {
  System.out.println(arg.toString());
}

public void testArgumentNonNull() {
  // 错误：参数声明为 @NonNull，但传入的是 null。
  argumentNonNull(null);
}
```

Checker Framework 对使用 Spring Framework 5.0 以上的用户非常有用，因为 Spring 提供了内置的空值检测注解，且能够被 Checker Framework 支持。一方面我们无需再引入额外的 Jar 包，更重要的是 Spring Framework 代码本身就使用了这些注解，这样我们在调用它的 API 时就能有效地处理空值了。举例来说，`StringUtils` 类里可以传入空值的函数、以及会返回空值的函数都添加了 `@Nullable` 注解，而未添加的方法则继承了整个框架的 `@NonNull` 注解，因此，下列代码中的空指针异常就可以被 Checker Framework 检测到了：

```java
// 这是 spring-core 中定义的类和方法
public abstract class StringUtils {
  // str 参数继承了全局的 @NonNull 注解
  public static String capitalize(String str) {}

  @Nullable
  public static String getFilename(@Nullable String path) {}
}

// 错误：参数声明为 @NonNull，但传入的是 null。
StringUtils.capitalize(null);

String filename = StringUtils.getFilename("/path/to/file");
// 错误：filename 可能为空。
System.out.println(filename.length());
```

## `Optional` 类型

Java 8 引入了 `Optional<T>` 类型，我们可以用它来对函数的返回值进行包装。这种方式的优点是可以明确定义该方法是有可能返回空值的，因此调用方必须做好相应处理，这样也就不会引发空指针异常。但是，也不可避免地需要编写更多代码，而且会产生很多垃圾对象，增加 GC 的压力，因此在使用时需要酌情考虑。


```java
Optional<String> opt;

// 创建
opt = Optional.empty();
opt = Optional.of("text");
opt = Optional.ofNullable(null);

// 判断并读取
if (opt.isPresent()) {
  opt.get();
}

// 默认值
opt.orElse("default");
opt.orElseGet(() -> "default");
opt.orElseThrow(() -> new NullPointerException());

// 相关操作
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

方法的链式调用很容易引发空指针异常，但如果返回值都用 `Optional` 包装起来，就可以用 `flatMap` 方法来实现安全的链式调用了：

```java
String zipCode = getUser()
    .flatMap(User::getAddress)
    .flatMap(Address::getZipCode)
    .orElse("");
```

Java 8 [Stream API][7] 同样使用了 `Optional` 作为返回类型：

```java
stringList.stream().findFirst().orElse("default");
stringList.stream()
    .max(Comparator.naturalOrder())
    .ifPresent(System.out::println);
```

此外，Java 8 还针对基础类型提供了单独的 `Optional` 类，如 `OptionalInt`、`OptionalDouble` 等，在性能要求比较高的场景下很适用。

## 其它 JVM 语言中的空指针异常

Scala 语言中的 [`Option`][8] 类可以对标 Java 8 的 `Optional`。它有两个子类型，`Some` 表示有值，`None` 表示空。

```scala
val opt: Option[String] = Some("text")
opt.getOrElse("default")
```

除了使用 `Option#isEmpty` 判断，还可以使用 Scala 的模式匹配：

```scala
opt match {
  case Some(text) => println(text)
  case None => println("default")
}
```

Scala 的集合处理函数库非常强大，`Option` 则可直接作为集合进行操作，如 `filer`、`map`、以及列表解析（for-comprehension）：

```scala
opt.map(_.trim).filter(_.length > 0).map(_.toUpperCase).getOrElse("DEFAULT")
val upper = for {
  text <- opt
  trimmed <- Some(text.trim())
  upper <- Some(trimmed) if trimmed.length > 0
} yield upper
upper.getOrElse("DEFAULT")
```

Kotlin 使用了另一种方式，用户在定义变量时就需要明确区分 [可空和不可空类型][9]。当可空类型被使用时，就必须进行空值检测。

```kotlin
var a: String = "text"
a = null // 错误：无法将 null 赋值给非空 String 类型。

val b: String? = "text"
// 错误：操作可空类型时必须使用安全操作符（?.）或强制忽略（!!.）。
println(b.length)

val l: Int? = b?.length // 安全操作
b!!.length // 强制忽略，可能引发空值异常
```

Kotlin 的特性之一是与 Java 的可互操作性，但 Kotlin 编译器无法知晓 Java 类型是否为空，这就需要在 Java 代码中使用注解了，而 Kotlin 支持的 [注解][9] 也非常广泛。Spring Framework 5.0 起原生支持 Kotlin，其空值检测也是通过注解进行的，使得 Kotlin 可以安全地调用 Spring Framework 的所有 API。

## 结论

在以上这些方案中，我比较推荐使用注解来预防空指针异常，因为这种方式十分有效，对代码的侵入性也较小。所有的公共 API 都应该使用 `@Nullable` 和 `@NonNull` 进行注解，这样就能强制调用方对空指针异常进行预防，让我们的程序更为健壮。

## 参考资料

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
