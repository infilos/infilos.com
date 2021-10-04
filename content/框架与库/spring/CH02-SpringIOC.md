---
type: docs
title: "CH02-Spring IOC"
linkTitle: "CH02-Spring IOC"
weight: 2
---

<!-- toc -->

## 基本概念

IoC（Inverse of Control:控制反转）是一种**设计思想**，就是 **将原本在程序中手动创建对象的控制权，交由Spring框架来管理**。 IoC 在其他语言中也有应用，并非 Spring 特有。 **IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个Map（key，value），Map 中存放的是各种对象**。

将对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。 **IoC 容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。** 在实际项目中一个 Service 类可能有几百甚至上千个类作为它的底层，假如我们需要实例化这个 Service，你可能要每次都要搞清这个 Service 所有底层类的构造函数，这可能会把人逼疯。如果利用 IoC 的话，你只需要配置好，然后在需要的地方引用就行了，这大大增加了项目的可维护性且降低了开发难度。

Spring IOC 通过引入 xml 配置，由 IOC 容器来管理对象的生命周期，依赖关系等。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210503105756.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

从图中可以看出，我们以前获取两个有依赖关系的对象，要用 set 方法，而用容器之后，它们之间的关系就由容器来管理。

## 什么是 Spring IOC 容器？

Spring 框架的核心是 Spring 容器。容器创建对象，将它们装配在一起，配置它们并管理它们的完整生命周期。Spring 容器使用**依赖注入**来管理组成应用程序的组件。容器通过读取提供的配置元数据来接收对象进行实例化，配置和组装的指令。该元数据可以通过 XML，Java 注解或 Java 代码提供。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210503105847.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

## 什么是依赖注入？

**依赖注入（DI,Dependency Injection）是在编译阶段尚未知所需的功能是来自哪个的类的情况下，将其他对象所依赖的功能对象实例化的模式**。这就需要一种机制用来激活相应的组件以提供特定的功能，所以**依赖注入是控制反转的基础**。否则如果在组件不受框架控制的情况下，框架又怎么知道要创建哪个组件？

依赖注入有以下三种实现方式：

1. 构造器注入
2. Setter方法注入（属性注入）
3. 接口注入

## Spring 中有多少种 IOC 容器？

在 Spring IOC 容器读取 Bean 配置创建 Bean 实例之前，必须对它进行实例化。只有在容器实例化后， 才可以从 IOC 容器里获取 Bean 实例并使用。

Spring 提供了两种类型的 IOC 容器实现

- **BeanFactory**：IOC 容器的基本实现
- **ApplicationContext**：提供了更多的高级特性，是 BeanFactory 的子接口

BeanFactory 是 Spring 框架的基础设施，面向 Spring 本身；ApplicationContext 面向使用 Spring 框架的开发者，几乎所有的应用场合都直接使用 ApplicationContext 而非底层的 BeanFactory；

无论使用何种方式，配置文件是相同的。

## BeanFactory

BeanFactory，从名字上可以看出来它是 bean 的工厂，它负责生产和管理各个 bean 实例。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210503110013.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

大概了解下这里提到的几个类：

- **ListableBeanFactory**：这个 Listable 的意思就是，通过这个接口，我们可以获取多个 Bean，大家看源码会发现，最顶层 BeanFactory 接口的方法都是获取单个 Bean 的。
- **HierarchicalBeanFactory**：Hierarchical 单词本身已经能说明问题了，也就是说我们可以在应用中起多个 BeanFactory，然后可以将各个 BeanFactory 设置为父子关系。
- **AutowireCapableBeanFactory**： 这个名字中的 Autowire 大家都非常熟悉，它就是用来自动装配 Bean 用的，但是仔细看上图，ApplicationContext 并没有继承它，不过不用担心，不使用继承，不代表不可以使用组合，如果你看到 ApplicationContext 接口定义中的最后一个方法 getAutowireCapableBeanFactory() 就知道了。
- **ConfigurableListableBeanFactory** ：也是一个特殊的接口，看图，特殊之处在于它继承了第二层所有的三个接口，而 ApplicationContext 没有。这点之后会用到。

