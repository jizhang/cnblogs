---
layout: post
title: "Java 反射机制"
date: 2014-01-25 09:42
comments: true
categories: Programming
tags: [translation]
published: true
---

原文：http://www.programcreek.com/2013/09/java-reflection-tutorial/

什么是反射？它有何用处？

## 1. 什么是反射？

“反射（Reflection）能够让运行于JVM中的程序检测和修改运行时的行为。”这个概念常常会和内省（Introspection）混淆，以下是这两个术语在Wikipedia中的解释：

1. 内省用于在运行时检测某个对象的类型和其包含的属性；
2. 反射用于在运行时检测和修改某个对象的结构及其行为。

从他们的定义可以看出，内省是反射的一个子集。有些语言支持内省，但并不支持反射，如C++。

![反射和内省](http://www.programcreek.com/wp-content/uploads/2013/09/reflection-introspection-650x222.png)

<!-- more -->

内省示例：`instanceof`运算符用于检测某个对象是否属于特定的类。

```java
if (obj instanceof Dog) {
    Dog d = (Dog) obj;
    d.bark();
}
```

反射示例：`Class.forName()`方法可以通过类或接口的名称（一个字符串或完全限定名）来获取对应的`Class`对象。`forName`方法会触发类的初始化。

```java
// 使用反射
Class<?> c = Class.forName("classpath.and.classname");
Object dog = c.newInstance();
Method m = c.getDeclaredMethod("bark", new Class<?>[0]);
m.invoke(dog);
```

在Java中，反射更接近于内省，因为你无法改变一个对象的结构。虽然一些API可以用来修改方法和属性的可见性，但并不能修改结构。

## 2. 我们为何需要反射？

反射能够让我们：

* 在运行时检测对象的类型；
* 动态构造某个类的对象；
* 检测类的属性和方法；
* 任意调用对象的方法；
* 修改构造函数、方法、属性的可见性；
* 以及其他

反射是框架中常用的方法。

例如，[JUnit](http://www.programcreek.com/2012/02/junit-tutorial-2-annotations/)通过反射来遍历包含 *@Test* 注解的方法，并在运行单元测试时调用它们。（[这个连接](http://www.programcreek.com/2012/02/junit-tutorial-2-annotations/)中包含了一些JUnit的使用案例）

对于Web框架，开发人员在配置文件中定义他们对各种接口和类的实现。通过反射机制，框架能够快速地动态初始化所需要的类。

例如，Spring框架使用如下的配置文件：

```xml
<bean id="someID" class="com.programcreek.Foo">
    <property name="someField" value="someValue" />
</bean>
```

当Spring容器处理&lt;bean&gt;元素时，会使用`Class.forName("com.programcreek.Foo")`来初始化这个类，并再次使用反射获取&lt;property&gt;元素对应的`setter`方法，为对象的属性赋值。

Servlet也会使用相同的机制：

```xml
<servlet>
    <servlet-name>someServlet</servlet-name>
    <servlet-class>com.programcreek.WhyReflectionServlet</servlet-class>
<servlet>
```

## 3. 如何使用反射？

让我们通过几个典型的案例来学习如何使用反射。

示例1：获取对象的类型名称。

```java
package myreflection;
import java.lang.reflect.Method;

public class ReflectionHelloWorld {
    public static void main(String[] args){
        Foo f = new Foo();
        System.out.println(f.getClass().getName());			
    }
}

class Foo {
    public void print() {
        System.out.println("abc");
    }
}
```

输出：

```text
myreflection.Foo
```

示例2：调用未知对象的方法。

在下列代码中，设想对象的类型是未知的。通过反射，我们可以判断它是否包含`print`方法，并调用它。

```java
package myreflection;
import java.lang.reflect.Method;

public class ReflectionHelloWorld {
    public static void main(String[] args){
        Foo f = new Foo();

        Method method;
        try {
            method = f.getClass().getMethod("print", new Class<?>[0]);
            method.invoke(f);
        } catch (Exception e) {
            e.printStackTrace();
        }           
    }
}

class Foo {
    public void print() {
        System.out.println("abc");
    }
}
```

```text
abc
```

示例3：创建对象

```java
package myreflection;

public class ReflectionHelloWorld {
    public static void main(String[] args){
        // 创建Class实例
        Class<?> c = null;
        try{
            c=Class.forName("myreflection.Foo");
        }catch(Exception e){
            e.printStackTrace();
        }

        // 创建Foo实例
        Foo f = null;

        try {
            f = (Foo) c.newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }   

        f.print();
    }
}

class Foo {
    public void print() {
        System.out.println("abc");
    }
}
```

示例4：获取构造函数，并创建对象。

```java
package myreflection;

import java.lang.reflect.Constructor;

public class ReflectionHelloWorld {
    public static void main(String[] args){
        // 创建Class实例
        Class<?> c = null;
        try{
            c=Class.forName("myreflection.Foo");
        }catch(Exception e){
            e.printStackTrace();
        }

        // 创建Foo实例
        Foo f1 = null;
        Foo f2 = null;

        // 获取所有的构造函数
        Constructor<?> cons[] = c.getConstructors();

        try {
            f1 = (Foo) cons[0].newInstance();
            f2 = (Foo) cons[1].newInstance("abc");
        } catch (Exception e) {
            e.printStackTrace();
        }   

        f1.print();
        f2.print();
    }
}

class Foo {
    String s;

    public Foo(){}

    public Foo(String s){
        this.s=s;
    }

    public void print() {
        System.out.println(s);
    }
}
```

```text
null
abc
```

此外，你可以通过`Class`实例来获取该类实现的接口、父类、声明的属性等。

示例5：通过反射来修改数组的大小。

```java
package myreflection;

import java.lang.reflect.Array;

public class ReflectionHelloWorld {
    public static void main(String[] args) {
        int[] intArray = { 1, 2, 3, 4, 5 };
        int[] newIntArray = (int[]) changeArraySize(intArray, 10);
        print(newIntArray);

        String[] atr = { "a", "b", "c", "d", "e" };
        String[] str1 = (String[]) changeArraySize(atr, 10);
        print(str1);
    }

    // 修改数组的大小
    public static Object changeArraySize(Object obj, int len) {
        Class<?> arr = obj.getClass().getComponentType();
        Object newArray = Array.newInstance(arr, len);

        // 复制数组
        int co = Array.getLength(obj);
        System.arraycopy(obj, 0, newArray, 0, co);
        return newArray;
    }

    // 打印
    public static void print(Object obj) {
        Class<?> c = obj.getClass();
        if (!c.isArray()) {
            return;
        }

        System.out.println("\nArray length: " + Array.getLength(obj));

        for (int i = 0; i < Array.getLength(obj); i++) {
            System.out.print(Array.get(obj, i) + " ");
        }
    }
}
```

输出：

```text
Array length: 10
1 2 3 4 5 0 0 0 0 0
Array length: 10
a b c d e null null null null null
```

## 总结

上述示例代码仅仅展现了Java反射机制很小一部分的功能。如果你觉得意犹未尽，可以前去阅读[官方文档](http://docs.oracle.com/javase/tutorial/reflect/)。

参考资料：

1. http://en.wikipedia.org/wiki/Reflection_(computer_programming)
2. http://docs.oracle.com/javase/tutorial/reflect/
