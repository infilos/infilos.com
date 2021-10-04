---
type: docs
title: "CH01-IOC"
linkTitle: "CH01-IOC"
weight: 1
---

<!-- toc -->

## IOC 是什么

**IoC，是 Inversion of Control 的缩写，即控制反转。\**他还有一个别名叫\**依赖注入**（Dependency Injection）有些资料也称依赖注入是IOC的一种常见方式。

IoC 不是什么技术，而是一种设计思想。在 Java 开发中，IoC 意味着将你设计好的对象交给容器控制，而不是传统的在你的对象内部直接控制。如何理解 Ioc 呢？理解 Ioc 的关键是要明确“谁控制谁，控制什么，为何是反转（有反转就应该有正转了），哪些方面反转了”，那我们来深入分析一下：

- **谁控制谁，控制什么：**传统 JavaSE 程序设计，我们直接在对象内部通过 new 进行创建对象，是程序主动去创建依赖对象；而 IoC 是有专门一个容器来创建这些对象，即由 IoC 容器来控制对象的创建；谁控制谁？当然是 IoC 容器控制了对象；控制什么？那就是主要控制了外部资源获取（不只是对象包括比如文件等）。
- **为何是反转，哪些方面反转了：**有反转就有正转，传统应用程序是由我们自己在对象中主动控制去直接获取依赖对象，也就是正转；而反转则是由容器来帮忙创建及注入依赖对象；为何是反转？因为由容器帮我们查找及注入依赖对象，对象只是被动的接受依赖对象，所以是反转；哪些方面反转了？依赖对象的获取方式被反转了。

## IOC 能做什么

IoC 不是一种技术，只是一种思想，一个重要的面向对象编程的法则，它能指导我们如何设计出松耦合、更优良的程序。传统应用程序都是由我们在类内部主动创建依赖对象，从而导致类与类之间高耦合，难于测试；有了 IoC 容器后，把创建和查找依赖对象的控制权交给了容器，由容器进行注入组合对象，所以对象与对象之间是松散耦合，这样也方便测试，利于功能复用，更重要的是使得程序的整个体系结构变得非常灵活。

其实 IoC 对编程带来的最大改变不是从代码上，而是从思想上，发生了“主从换位”的变化。应用程序原本是老大，要获取什么资源都是主动出击，但是在 IoC/DI 思想中，应用程序就变成被动的了，被动的等待 IoC 容器来创建并注入它所需要的资源了。

IoC 很好的体现了面向对象设计法则之一—— 好莱坞法则：“Don't call us,we will call you”；即由 IoC 容器帮对象找相应的依赖对象并注入，而不是由对象主动去找。

### 依赖注入

DI，是 **Dependency Injection** 的缩写，即依赖注入。依赖注入是 IoC 的最常见形式。

容器全权负责的组件的装配，它会把符合依赖关系的对象通过 JavaBean 属性或者构造函数传递给需要的对象。

DI 是组件之间依赖关系由容器在运行期决定，形象的说，即由容器动态的将某个依赖关系注入到组件之中。依赖注入的目的并非为软件系统带来更多功能，而是为了提升组件重用的频率，并为系统搭建一个灵活、可扩展的平台。通过依赖注入机制，我们只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑，而不需要关心具体的资源来自何处，由谁实现。

理解 DI 的关键是：“谁依赖谁，为什么需要依赖，谁注入谁，注入了什么”，那我们来深入分析一下：

- **谁依赖于谁：**当然是应用程序依赖于 IoC 容器；
- **为什么需要依赖：**应用程序需要 IoC 容器来提供对象需要的外部资源；
- **谁注入谁：**很明显是 IoC 容器注入应用程序某个对象，应用程序依赖的对象；
- **注入了什么**：就是注入某个对象所需要的外部资源（包括对象、资源、常量数据）。

### IOC 与 DI

其实它们是同一个概念的不同角度描述，由于控制反转概念比较含糊（可能只是理解为容器控制对象这一个层面，很难让人想到谁来维护对象关系），所以 2004 年大师级人物 Martin Fowler 又给出了一个新的名字：“依赖注入”，相对 IoC 而言，“依赖注入”明确描述了“被注入对象依赖 IoC 容器配置依赖对象”。

>  [Martin Fowler—Inversion of Control Containers and the Dependency Injection pattern](http://www.martinfowler.com/articles/injection.html)

### IOC 容器

IoC 容器就是具有依赖注入功能的容器。IoC 容器负责实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。应用程序无需直接在代码中 new 相关的对象，应用程序由 IoC 容器进行组装。在 Spring 中 BeanFactory 是 IoC 容器的实际代表者。

Spring IoC 容器如何知道哪些是它管理的对象呢？这就需要配置文件，Spring IoC 容器通过读取配置文件中的配置元数据，通过元数据对应用中的各个对象进行实例化及装配。一般使用基于 xml 配置文件进行配置元数据，而且 Spring 与配置文件完全解耦的，可以使用其他任何可能的方式进行配置元数据，比如注解、基于 java 文件的、基于属性文件的配置都可以

那 Spring IoC 容器管理的对象叫什么呢？

### Bean

JavaBean 是一种JAVA语言写成的可重用组件。为写成JavaBean，类必须是具体的和公共的，并且具有**无参数的构造器**。JavaBean 通过提供符合一致性设计模式的公共方法（getter / setter 方法）将内部域暴露成员属性。众所周知，属性名称符合这种模式，其他Java 类可以通过自省机制发现和操作这些JavaBean 的属性。

一个javaBean由三部分组成：**属性、方法、事件**

JavaBean的任务就是: “Write once, run anywhere, reuse everywhere”，即“一次性编写，任何地方执行，任何地方重用”。

由 IoC 容器管理的那些组成你应用程序的对象我们就叫它 Bean。Bean 就是由 Spring 容器初始化、装配及管理的对象，除此之外，bean 就与应用程序中的其他对象没有什么区别了。

## IOC 容器

### 核心接口

`org.springframework.beans` 和 `org.springframework.context` 是 IoC 容器的基础。

在 Spring 中，有两种 IoC 容器：`BeanFactory` 和 `ApplicationContext`。

- `BeanFactory`：Spring 实例化、配置和管理对象的最基本接口。
- `ApplicationContext`：BeanFactory 的子接口。它还扩展了其他一些接口，以支持更丰富的功能，如：国际化、访问资源、事件机制、更方便的支持 AOP、在 web 应用中指定应用层上下文等。

实际开发中，更推荐使用 `ApplicationContext` 作为 IoC 容器的操作入口，因为它的功能远多于 `FactoryBean`。

常见 `ApplicationContext` 实现：

- **ClassPathXmlApplicationContext**：`ApplicationContext` 的实现，从 classpath 获取配置文件；
  - `new ClassPathXmlApplicationContext("classpath.xml");`
- **FileSystemXmlApplicationContext**：`ApplicationContext` 的实现，从文件系统获取配置文件。
  - `new FileSystemXmlApplicationContext("fileSystemConfig.xml");`

### 应用流程

使用 IoC 容器可分为三步骤：

1. 配置元数据：需要配置一些元数据来告诉 Spring，你希望容器如何工作，具体来说，就是如何去初始化、配置、管理 JavaBean 对象。
2. 实例化容器：由 IoC 容器解析配置的元数据。IoC 容器的 Bean Reader 读取并解析配置文件，根据定义生成 BeanDefinition 配置元数据对象，IoC 容器根据 BeanDefinition 进行实例化、配置及组装 Bean。
3. 使用容器：由客户端实例化容器，获取需要的 Bean。

### 配置元数据

> **元数据（Metadata）** 又称中介数据、中继数据，为描述数据的数据（data about data），主要是描述数据属性（property）的信息。

配置元数据的方式：

- **基于 xml 配置**：Spring 的传统配置方式。在 `<beans>` 标签中配置元数据内容。

  缺点是当 JavaBean 过多时，产生的配置文件足以让你眼花缭乱。

- **基于注解配置**：Spring2.5 引入。可以大大简化你的配置。

- **基于 Java 配置**：可以使用 Java 类来定义 JavaBean 。

  为了使用这个新特性，需要用到 `@Configuration` 、`@Bean` 、`@Import` 和 `@DependsOn` 注解。

### Bean 概述

一个 Spring 容器管理一个或多个 bean。 这些 bean 根据你配置的元数据（比如 xml 形式）来创建。 Spring IoC 容器本身，并不能识别你配置的元数据。为此，要将这些配置信息转为 Spring 能识别的格式——BeanDefinition 对象。

#### 命名 Bean

指定 id 和 name 属性不是必须的。 Spring 中，并非一定要指定 id 和 name 属性。实际上，Spring 会自动为其分配一个特殊名。 如果你需要引用声明的 bean，这时你才需要一个标识。官方推荐驼峰命名法来命名。

#### 支持别名

可能存在这样的场景，不同系统中对于同一 bean 的命名方式不一样。 为了适配，Spring 支持 `<alias>` 为 bean 添加别名的功能。

```xml
<alias name="subsystemA-dataSource" alias="subsystemB-dataSource"/>
<alias name="subsystemA-dataSource" alias="myApp-dataSource" />
```

#### 实例化 Bean

**构造器方式**

```xml
<bean id="exampleBean" class="examples.ExampleBean"/>
```

**静态工厂方法**

### 依赖

依赖注入 依赖注入有两种主要方式：

- 构造器注入
- Setter 注入 构造器注入有可能出现循环注入的错误。如：

```java
class A {
	public A(B b){}
}
class B {
	public B(A a){}
}
```

**依赖和配置细节** 使用 depends-on Lazy-initialized Bean 自动装配 方法注入。

## IoC 容器配置

IoC 容器的配置有三种方式：

- 基于 xml 配置
- 基于注解配置
- 基于 Java 配置

作为 Spring 传统的配置方式，xml 配置方式一般为大家所熟知。

如果厌倦了 xml 配置，Spring 也提供了注解配置方式或 Java 配置方式来简化配置。

### Xml 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
         http://www.springframework.org/schema/beans/spring-beans.xsd">
  <import resource="resource1.xml" />
  <bean id="bean1" class=""></bean>
  <bean id="bean2" class=""></bean>
  <bean name="bean2" class=""></bean>

  <alias alias="bean3" name="bean2"/>
  <import resource="resource2.xml" />
</beans>
```

标签说明：

- `<beans>` 是 Spring 配置文件的根节点。
- `<bean>` 用来定义一个 JavaBean。`id` 属性是它的标识，在文件中必须唯一；`class` 属性是它关联的类。
- `<alias>` 用来定义 Bean 的别名。
- `<import>` 用来导入其他配置文件的 Bean 定义。这是为了加载多个配置文件，当然也可以把这些配置文件构造为一个数组（new String[] {“config1.xml”, config2.xml}）传给 `ApplicationContext` 实现类进行加载多个配置文件，那一个更适合由用户决定；这两种方式都是通过调用 Bean Definition Reader 读取 Bean 定义，内部实现没有任何区别。`<import>` 标签可以放在 `<beans>` 下的任何位置，没有顺序关系。

#### 实例化容器

实例化容器的过程： 定位资源（XML 配置文件） 读取配置信息(Resource) 转化为 Spring 可识别的数据形式（BeanDefinition）

```java
ApplicationContext context =
      new ClassPathXmlApplicationContext(new String[] {"services.xml", "daos.xml"});
```

组合 xml 配置文件 配置的 Bean 功能各不相同，都放在一个 xml 文件中，不便管理。 Java 设计模式讲究职责单一原则。配置其实也是如此，功能不同的 JavaBean 应该被组织在不同的 xml 文件中。然后使用 import 标签把它们统一导入。

```xml
<import resource="classpath:spring/applicationContext.xml"/>
<import resource="/WEB-INF/spring/service.xml"/>
```

#### 使用容器

使用容器的方式就是通过`getBean`获取 IoC 容器中的 JavaBean。 Spring 也有其他方法去获得 JavaBean，但是 Spring 并不推荐其他方式。

```java
// create and configure beans
ApplicationContext context =
new ClassPathXmlApplicationContext(new String[] {"services.xml", "daos.xml"});
// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);
// use configured instance
List<String> userList = service.getUsernameList();
```

### 注解配置

Spring2.5 引入了注解。 **优点**：大大减少了配置，并且可以使配置更加精细——类，方法，字段都可以用注解去标记。 **缺点**：使用注解，不可避免产生了侵入式编程，也产生了一些问题。

- 你需要将注解加入你的源码并编译它；
- 注解往往比较分散，不易管控。

> 注：spring 中，先进行注解注入，然后才是 xml 注入，因此如果注入的目标相同，后者会覆盖前者。

#### 启动注解

Spring 默认是不启用注解的。如果想使用注解，需要先在 xml 中启动注解。 启动方式：在 xml 中加入一个标签，很简单吧。

```xml
<context:annotation-config/>
```

> 注：`<context:annotation-config/>` 只会检索定义它的上下文。什么意思呢？就是说，如果你 为 DispatcherServlet 指定了一个`WebApplicationContext`，那么它只在 controller 中查找`@Autowired`注解，而不会检查其它的路径。

####  Spring 注解

- **`@Required`**：只能用于修饰 bean 属性的 setter 方法。受影响的 bean 属性必须在配置时被填充在 xml 配置文件中，否则容器将抛出`BeanInitializationException`。
- **`@Autowired`**：可用于修饰属性、setter 方法、构造方法。
  - 可以使用 JSR330 的注解`@Inject`来替代`@Autowired`。
- **`@Qualifier`**：如果发现有多个候选的 bean 都符合修饰类型，指定 bean 名称来锁定真正需要的那个 bean。
- JSR 250 注解
  - `@Resource`
  - **`@PostConstruct` 和 `@PreDestroy`**
- JSR 330 注解
  - `@Inject`

### Java 配置

基于 Java 配置 Spring IoC 容器，实际上是 Spring 允许用户定义一个类，在这个类中去管理 IoC 容器的配置。

为了让 Spring 识别这个定义类为一个 Spring 配置类，需要用到两个注解：`@Configuration` 和 `@Bean`。

如果你熟悉 Spring 的 xml 配置方式，你可以将 `@Configuration` 等价于 `<beans>` 标签；将 `@Bean` 等价于 `<bean>` 标签。

#### @Bean

- @Bean 的修饰目标只能是方法或注解。

- @Bean 只能定义在@Configuration 或@Component 注解修饰的类中。
- @Configuration 类允许在同一个类中通过@Bean 定义内部 bean 依赖。
- 声明一个 bean，只需要在 bean 属性的 set 方法上标注@Bean 即可。

```java
@Configuration
public class AnnotationConfiguration {
  
    @Bean
    public Job getPolice() {
        return new Police();
    }
}

public interface Job {
    String work();
}

@Component("police")
public class Police implements Job {
    @Override
    public String work() {
        return "抓罪犯";
    }
}
```

#### @Configuration

`@Configuration` 是一个类级别的注解，用来标记被修饰类的对象是一个`BeanDefinition`。

`@Configuration` 声明 bean 是通过被 `@Bean` 修饰的公共方法。此外，`@Configuration` 允许在同一个类中通过 `@Bean` 定义内部 bean 依赖。

```java
@Configuration
public class AppConfig {
    @Bean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

