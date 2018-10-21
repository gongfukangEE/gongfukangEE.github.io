---
layout: post
title:  "设计模式[1] -- 单例模式"
categories: 设计模式
tags:  单例模式
author: G.Fukang
---

* content
{:toc}
单例模式（创建型模式）实现：懒汉式、饿汉式、静态内部类式、枚举式


## 核心作用

**保证一个类只有一个实例，并且提供一个访问该实例的全局访问点**

## 应用场景

- `Windows`的任务管理器和回收站都是典型的单例模式，不论打开多少次，只能有且仅有一个实例
- 网站计数器和应用程序的日志应用都是单例模式
- 数据库连接池的设计一般也是采用单例模式，因为数据库连接是一种数据库资源
- 操作系统的文件系统也是单例模式，一个操作系统只能有一个文件系统
- 在`Spring`中，每个`Bean`默认就是单例的，这样的优点是`Spring`容器可以管理
- 在`servlet`编程中，每个`servlet`也是单例
- 在`Spring MVC`框架中，控制对象也是单例模式

## 优点

- 由于单例模式只生成一个实例，减少了系统性能的开销，当一个对象的产生需要比较多的资源的时，如读取配置、产生其他依赖对象时，则可以通过在应用启动的时候直接产生一个单例对象，然后永久驻留内存的方式来解决
- 单例模式可以在系统设置全局的访问点，优化共享资源访问，例如可以设计一个单例，负责所有数据表的处理

## 常见实现方式

### 懒汉式

**懒汉式单例模式**：线程安全，加载延迟，真正在使用的时候才加载，提高了资源利用率，但是，每次调用`getinstance()`方法都要同步，并发效率较低。

代码片段

```java
 /**
 * @Auther gongfukang
 * @Date 2018/1/31 21:02
 * 懒汉式单例模式
 */
public class Singleton02 {

    //类初始化时，不初始化这个对象（延时加载，真正用的时候再创建）。
    private static Singleton02 instance;

    //私有化构造器
    private Singleton02(){};

    //方法同步，调用效率低！
    public static synchronized Singleton02 getInstance(){
        if(instance==null)
            instance=new Singleton02();
        return instance;
    }
}
```

​

### 饿汉式

**饿汉式单例模式**：线程安全，不能延时加载，调用效率高，`static`变量会在类装载的时候初始化，此时也不会涉及多个线程对象访问该对象的问题。虚拟机保证只会装载一次该类，肯定不会发生并发访问的问题，因此可以省略`synchronized`关键字。但是如果之加载本类，而不是调用`getinstance()`就会造成资源浪费

代码片段

```java
/**
 * @Auther gongfukang
 * @Date 2018/1/31 20:58
 * 饿汉式单例模式
 */
public class Singleton01 {

    //类初始化时，立即加载这个对象（没有延时加载的优势）。加载类时，天然的是线程安全的！
    private static Singleton01 instance=new Singleton01();

    private Singleton01(){}

    //方法没有同步，调用效率高！
    public static Singleton01 getInstance(){
        return instance;
    }
}
```

### 静态内部类式

**静态内部类式单例模式**：线程安全，调用效率高，并且实现了延时加载。外部没有`static`属性，不会像饿汉式那样立即加载对象，只有真正调用`getInstance()`才会加载静态内部类，加载类时是线程安全的。`instance`是`static final`类型，保证内存中只有这样一个实例存在，而且只能被赋值一次，从而保证了线程安全，兼并了并发高效调用和延迟加载的优势

代码片段

```java
/**
 * @Auther gongfukang
 * @Date 2018/1/31 22:01
 * 静态内部类实现单例模式
 */
public class Singleton04 {
    private static class SingletonClassInstance{
        private static final Singleton04 instance=new Singleton04();
    }

    private Singleton04(){}

    public static Singleton04 getInstance(){
        return SingletonClassInstance.instance;
    }
}
```

### 枚举式

**枚举式实现单例模式**：实现简单，无延迟加载

代码片段

```java
/**
 * @Auther gongfukang
 * @Date 2018/1/31 22:04
 * 枚举实现单例模式
 */
public enum  Singleton05 {

    //这个枚举元素本身就是单例对象
    INSTANCE;

    //添加自己需要的操作
    public void singletonOperation(){

    }
}
```

