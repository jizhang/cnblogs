---
layout: post
title: "使用 Spring AOP 向领域模型注入依赖"
date: 2015-09-12 22:03
comments: true
categories: [Programming]
tags: [spring, aop]
published: true
---

在[贫血领域模型](http://shzhangji.com/cnblogs/2015/09/05/anemic-domain-model/)这篇译文中，Martin阐述了这种“反模式”的症状和问题，并引用了领域驱动设计中的话来说明领域模型和分层设计之间的关系。对于Spring项目的开发人员来说，贫血领域模型十分常见：模型（或实体）仅仅包含对数据表的映射，通常是一组私有属性和公有getter/setter，所有的业务逻辑都写在服务层中，领域模型仅仅用来传递数据。为了编写真正的领域模型，我们需要将业务逻辑移至模型对象中，这就引出另一个问题：业务逻辑通常需要调用其他服务或模型，而使用`new`关键字或由JPA创建的对象是不受Spring托管的，也就无法进行依赖注入。解决这个问题的方法有很多，比较之后我选择使用面向切面编程来实现。

## 面向切面编程

面向切面编程，或[AOP](https://en.wikipedia.org/wiki/Aspect-oriented_programming)，是一种编程范式，和面向对象编程（[OOP](https://en.wikipedia.org/wiki/Object-oriented_programming)）互为补充。简单来说，AOP可以在不修改既有代码的情况下改变代码的行为。开发者通过定义一组规则，在特定的类方法前后增加逻辑，如记录日志、性能监控、事务管理等。这些逻辑称为切面（Aspect），规则称为切点（Pointcut），在调用前还是调用后执行称为通知（Before advice, After advice）。最后，我们可以选择在编译期将这些逻辑写入类文件，或是在运行时动态加载这些逻辑，这是两种不同的织入方式（Compile-time weaving, Load-time weaving）。

对于领域模型的依赖注入，我们要做的就是使用AOP在对象创建后调用Spring框架来注入依赖。幸运的是，[Spring AOP](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#aop)已经提供了`@Configurable`注解来帮助我们实现这一需求。

<!-- more -->

## Configurable注解

Spring应用程序会定义一个上下文容器，在该容器内创建的对象会由Spring负责注入依赖。对于容器外创建的对象，我们可以使用`@Configurable`来修饰类，告知Spring对这些类的实例也进行依赖注入。

假设有一个`Report`类（领域模型），其中一个方法需要解析JSON，我们可以使用`@Configurable`将容器内的`ObjectMapper`对象注入到类的实例中：

```java
@Entity
@Configurable(autowire = Autowire.BY_TYPE)
public class Report {

    @Id @GeneratedValue
    private Integer id;

    @Autowired @Transient
    private ObjectMapper mapper;

    public String render() {
        mapper.readValue(...);
    }

}
```

* `autowire`参数默认是`NO`，因此需要显式打开，否则只能使用XML定义依赖。`@Autowired`是目前比较推荐的注入方式。
* `@Transient`用于告知JPA该属性不需要进行持久化。你也可以使用`transient`关键字来声明，效果相同。
* 项目依赖中需要包含`spring-aspects`。如果已经使用了`spring-boot-starter-data-jpa`，则无需配置。
* 应用程序配置中需要加入`@EnableSpringConfigured`：

```java
@SpringBootApplication
@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)
@EnableSpringConfigured
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

* 在`src/main/resources`目录下，新建`META-INF/aop.xml`文件，用来限定哪些包会用到AOP。否则，AOP的织入操作会作用于所有的类（包括第三方类库），产生不必要的的报错信息。

```xml
<!DOCTYPE aspectj PUBLIC "-//AspectJ//DTD//EN" "http://www.eclipse.org/aspectj/dtd/aspectj.dtd">

<aspectj>
    <weaver>
        <include within="com.foobar..*"/>
    </weaver>
</aspectj>
```

## 运行时织入（Load-Time Weaving, LTW）

除了项目依赖和应用程序配置，我们还需要选择一种织入方式来使AOP生效。Spring AOP推荐的方式是运行时织入，并提供了一个专用的Jar包。运行时织入的原理是：当类加载器在读取类文件时，动态修改类的字节码。这一机制是从[JDK1.5](http://docs.oracle.com/javase/1.5.0/docs/api/java/lang/instrument/package-summary.html)开始提供的，需要使用`-javaagent`参数开启，如：

```bash
$ java -javaagent:/path/to/spring-instrument.jar -jar app.jar
```

在测试时发现，Spring AOP提供的这一Jar包对普通的类是有效果的，但对于使用`@Entity`修饰的类就没有作用了。因此，我们改用AspectJ提供的Jar包（可到[Maven中央仓库](http://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22org.aspectj%22%20AND%20a%3A%22aspectjweaver%22)下载）：

```bash
$ java -javaagent:/path/to/aspectjweaver.jar -jar app.jar
```

对于[Spring Boot](http://projects.spring.io/spring-boot/)应用程序，可以在Maven命令中加入以下参数：

```bash
$ mvn spring-boot:run -Drun.agent=/path/to/aspectjweaver.jar
```

此外，在使用AspectJ作为LTW的提供方后，会影响到Spring的事务管理，因此需要在应用程序配置中加入：

```java
@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)
```

## AnnotationBeanConfigurerAspect

到这里我们已经通过简单配置完成了领域模型的依赖注入，这背后都是Spring中的`AnnotationBeanConfigurerAspect`在做工作。我们不妨浏览一下精简后的源码：

[AnnotationBeanConfigurerAspect.aj](https://github.com/spring-projects/spring-framework/blob/master/spring-aspects/src/main/java/org/springframework/beans/factory/aspectj/AnnotationBeanConfigurerAspect.aj)

```aspectj
public aspect AnnotationBeanConfigurerAspect implements BeanFactoryAware {

	private BeanConfigurerSupport beanConfigurerSupport = new BeanConfigurerSupport();

	public void setBeanFactory(BeanFactory beanFactory) {
		this.beanConfigurerSupport.setBeanFactory(beanFactory);
	}

	public void configureBean(Object bean) {
		this.beanConfigurerSupport.configureBean(bean);
	}

	public pointcut inConfigurableBean() : @this(Configurable);

	declare parents: @Configurable * implements ConfigurableObject;

	public pointcut beanConstruction(Object bean) :
			initialization(ConfigurableObject+.new(..)) && this(bean);

	after(Object bean) returning :
		beanConstruction(bean) && inConfigurableBean() {
		configureBean(bean);
	}
}
```

* `.aj`文件是AspectJ定义的语言，增加了pointcut、after等关键字，用来定义切点、通知等；
* `inConfigurationBean`切点用于匹配使用`Configurable`修饰的类型；
* `declare parents`将这些类型声明为`ConfigurableObject`接口，从而匹配`beanConstruction`切点；
* `ConfigurableObject+.new(..)`表示匹配该类型所有的构造函数；
* `after`定义一个通知，表示对象创建完成后执行`configureBean`方法；
* 该方法会调用`BeanConfigurerSupport`来对新实例进行依赖注入。

## 其它方案

1. 将依赖作为参数传入。比如上文中的`render`方法可以定义为`render(ObjectMapper mapper)`。
2. 将`ApplicationContext`作为某个类的静态成员，领域模型通过这个引用来获取依赖。
3. 编写一个工厂方法，所有新建对象都要通过这个方法生成，进行依赖注入。
4. 如果领域模型大多从数据库获得，并且JPA的提供方是Hibernate，则可以使用它的拦截器功能进行依赖注入。

## 参考资料

* http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#aop-atconfigurable
* http://blog.igorstoyanov.com/2005/12/dependency-injection-or-service.html
* http://jblewitt.com/blog/?p=129
