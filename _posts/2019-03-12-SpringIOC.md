---
layout: post
title:  "浅析 Spring IOC 机制"
categories: [WEB开发, Spring, IOC]
tags:  Spring 
author: G.Fukang
---
浅析 Spring IOC 机制

# IOC 理论

IoC 全称为 `Inversion of Control`，翻译为 “控制反转”，它还有一个别名为 DI（`Dependency Injection`），即依赖注入，Spring 官方给出的定义为**所谓 IOC，就是由 Spring IOC 容器来负责对象的生命周期和对象之间的关系**。而理解好**控制反转**的关键在于回答以下四个问题：

1. 谁控制谁
2. 控制什么
3. 为何是反转
4. 哪些方面反转了

根据《Spring 揭秘》中给出的 IOC 示意图可以看出：

![](http://static.iocoder.cn/a9c6a574b85de4723729b674cb51436f)

在没有引入 IoC 的时候，被注入的对象直接依赖于被依赖的对象，有了 IoC 后，两者及其他们的关系都是通过 Ioc Service Provider 来统一管理维护的。被注入的对象需要什么，直接跟 IoC Service Provider 打声招呼，后者就会把相应的被依赖对象注入到被注入的对象中，从而达到 IoC Service Provider 为被注入对象服务的目的。**所以 IoC 就是这么简单！原来是需要什么东西自己去拿，现在是需要什么东西让别人（IoC Service Provider）送过来**

这样来看，回答上面四个问题就很简单了

1. **谁控制谁：**在传统的开发模式下，我们都是采用直接 new 一个对象的方式来创建对象，也就是说你依赖的对象直接由你自己控制，但是有了 IoC 容器后，则直接由 IoC 容器来控制。所以“谁控制谁”，当然是 IoC 容器控制对象
2. **控制什么：**控制对象
3. **为何是反转：**没有 IoC 的时候我们都是在自己对象中主动去创建被依赖的对象，这是正转。但是有了 IoC 后，所依赖的对象直接由 IoC 容器创建后注入到被注入的对象中，依赖的对象由原来的主动获取变成被动接受，所以是反转
4. **哪些方面反转了：**所依赖对象的获取被反转了

# IOC 注入形式

IOC Service Provider 为被注入对象提供被依赖对象也有如下几种方式：

**构造器注入：**对象构造完毕后可以直接使用

**setter 方法注入：**当前对象需要为所依赖的对象提供 setter 方法，可以在任何时候注入，比较灵活宽松

**接口注入：**需要被以来的对象实现不必要的接口，带有侵入性

# IOC 结构

Bean 配置信息首先定义了 Bean 的实现以及依赖关系，Spring 容器根据各种形式的 Bean 配置信息在容器内部建立 Bean 定义注册表；然后根据注册表加载、实例化 Bean，并建立 Bean 和 Bean 之间的依赖关系；最后将这些准备就绪的 Bean 放到 Bean 缓存池中，以供外层的应用程序进行调用。

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Spring/IOC.jpg)

#  IOC 实现

Spring 的 IOC 实现主要分为两个部分：容器的初始化阶段和加载 bean 阶段

####  容器初始化阶段

初始化的主要过程就是：Resource 定位、BeanDefinition 的载入和解析，BeanDefinition 注册

1. 资源文件定位

   Spring 提供了 Resource 和 ResourceLoader 来统一抽象整个资源及其定位，一般是在 ApplicationContext 的实现类里完成的，ResourceLoader 从存储介质中加载 Spring 配置信息，并使用 Resource 表示这个配置文件资源

2. 解析并装载

   这个流程就是将用户在配置文件中定义的 `<bean>` 表示成 IOC 容器内部的数据结构 BeaDefinition。

   ApplicationContext 完成资源文件定位后，将解析工作委托给 XmlBeanDefinitionReader 来完成，其中的 loadBeanDefinitions 方法负责读取 Resource，并将其解析成 BeanDefinition

3. 注册

   这个流程就是将上一个步骤得到的 BeanDefinition 的 name 和实例注入到 HashMap 中，IOC 容器就是通过这个 HashMap 来维护 BeanDefinition 的。

   Bean 的注册是在 BeanFactory 里完成的，通过 BeanDefinitionRegistry 接口对 Bean 进行注册。这里并没有完成依赖注入（Bean 创建），Bean 创建是发生在应用第一次调用 getBean 方法时，我们可以通过配置 `lazyinit = false` ，设置为 Bean 的依赖注入在容器初始化时完成

####  加载 Bean 阶段

1. 从缓存中获取 Bean

   若目标 Bean 在 Bean 缓存池中，则直接获取

2. 创建 Bean 实例对象

   如果缓存池中没有目标 Bean 或者该 Bean 不是单例的，则调用 createBeanInstance 方法创建 Bean 实例，其核心思想为确定 Bean 的构造函数和构造方法，使用**工厂方法**初始化，再**采用策略模式决定是使用反射或者 CGLIB 动态字节码实例化 Bean**；然后调用 populateBean 方法将属性值赋值给实例化的 Bean；最后调用 createBean 方法激活 Bean 中 init 方法来初始化 Bean

3. 从 Bean 实例中获取对象

   调用 getObjectForBeanInstance 方法根据传入的 bean 实例获取对象，返回给请求方

####  常用 IOC 容器

1. BeanFactory：类的通用工厂，它可以创建并管理各种类对象，Spring 称这些被创建和管理的 Java 对象为 Bean
2. ApplicationContext：由 BeanFactory 派生而来，提供了更多面向实际应用的功能，比如管理 message，事件发布，多资源加载，管理生命周期等。有三种实现：ClassPathXmlApplicationContext 、FileSystemXmlApplicationContext 、XmlWebApplicationContext ，分别从不同位置读取上下文

# Bean

####  Bean 分类

1. singleton：在 Spring 的 IoC 容器中只存在一个对象实例，所有该对象的引用都共享这个实例。Spring 容器只会创建该 bean 定义的唯一实例，这个实例会被保存到缓存中，并且对该bean的所有后续请求和引用都将返回该缓存中的对象实例。
2. prototype：为每一个bean请求提供一个实例
3. 其他：
   1. Request：每一次 HTTP 请求都会产生一个新的 Bean 实例
   2. Session：每一个的 Session 都会产生一个新的 Bean 实例
   3. Application：每一个 Web Application 都会产生一个新的 Bean

####  Bean 生命周期

1. 显式或隐式调用 BeanFactory 的 getBean 方法来请求某个实例对象的时候，就会触发 Bean 的实例化进程

2. Bean 实例化

   调用 createBeanInstance 方法返回一个 BeanWrapper 对象，然后通过默认实现类 BeanWrapperImpl 来对 Bean 进行属性注入（策略模式）

3. Bean 初始化

   1. 激活 Aware：检查当前 Bean 是否实现了一系列以 Aware 结尾的接口（实现了该接口的 Bean 具有被 Spring 容器通知的能力）
      1. BeanNameAware：将 Bean 对象定义的 BeanName 设置到当前实例对象中
      2. BeanClassLoaderAware：将当前 Bean 对象对应的 ClassLoader 注入到当前对象实例中
      3. BeanFactoryAware：BeanFactory 容器会将自身注入到当前对象实例中，这样当前对象就会拥有一个 BeanFactory 容器的引用
   2. BeanPostProcess 增强处理：BeanPostProcess 会对 Spring 容器提供的 Bean 对象在初始化阶段进行定制化修改，会在 init 方法前后进行前置增强和后置增强处理
   3. 检查 initializingBean （侵入性）和 init-method （反射）方法：这两种方法都可以对 Bean 进行初始化定制

4. 使用 Bean

   初始化后就可以使用 Bean 了，会将 Bean 放到 Spring 容器的 Bean 缓存池中，供使用者调用

5. 在调用完成后，Spring 容器会管理 Bean 后续的生命周期，如果 Bean 实现了 DisposableBean 接口或者配置了 init-method 方法，就会在对象销毁前执行销毁逻辑

## Bean 的转换过程

![](https://gitee.com/chenssy/blog-home/raw/master/image/201809/spring-201901311001.jpg)







