---
layout: post
title:  "浅析 Dubbo SPI 机制"
categories: [分布式, Dubbo, RPC]
tags:  分布式 Dubbo RPC 
author: G.Fukang
---
浅析 Dubbo SPI 机制

> 转载自 [Dubbo 源码导读 -- Dubbo SPI](http://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html) 笔者对其进行重新排版，并加入部分自己的理解

### SPI 机制

#### 什么是 SPI

如果提到 API 大家肯定都知道，但是 SPI 知道的人就少一点。简单来说，API 是给使用者使用的，SPI 是给拓展者使用的，一个好的开源框架，必须留一些拓展点，让参与者尽量黑盒拓展，而不是白盒修改代码，否则分支、质量、合并、冲突都难以管理，并且框架作者能做的的功能，拓展者也能做到。

如果从使用层面来说，**它是以一种服务发现机制，SPI 的本质是将接口实现类的全限定名配置在文件张，并由服务器加载读取配置文件，加载实现类，这样可以在运行时，动态为接口替换实现类**。其实这有有点像`IoC`的思想，将装配的控制权移到程序之外

#### SPI 简单实现

接口和具体实现类

```java
public interface Robot {
    void sayHello();
}
```

```java
public class OptimusPrime implements Robot{
    @Override
    public void sayHello() {
        System.out.println("OptimusPrime");
    }
}
```

```java
public class Bumblebee implements Robot{
    @Override
    public void sayHello() {
        System.out.println("Bumblebee");
    }
}
```

配置文件，放在 `META-INF/service/接口全限定名`

```java
com.gfk.spi.impl.OptimusPrime
com.gfk.spi.iml.Bumblebee
```

编写测试类

```java
@Test
public void test() throws Exception {
    ServiceLoader<Robot> serviceLoader = ServiceLoader.load(Robot.class);
    System.out.println("Java SPI")
        serviceLoader.forEach(Robot::sayHello);
}
/* ouput*/
Java SPI
OptimusPrime
Bumblebee
```

通过以上测试代码可以看出：改变配置文件，就能动态改变一个接口的实现类，这也类似于 Spring 的 IOC 通过配置文件动态注入不同的实现类。

当然，如果想增加一个新的实现类 `SayJapansesNameImpl`，但是预先包里没有这个实现类，只改配置文件，也不会有效果。但是可以利用动态字节码技术 `javassist`，在运行时动态生成 Java 类，就不需要预先把接口的实现类先放在包里。

### 从双亲委派模型到 JDK SPI

#### 双亲委派模型简单介绍

`Class`的`装载`大体上可以分为加载类、连接类、初始化三个阶段，在这三个阶段中，所有的`Class`都是由`ClassLoader`进行加载的，然后Java虚拟机负责连接、初始化等操作。也就是说，无法通过`ClassLoader`去改变类的连接和初始化行为。系统中的`ClassLoader`在协同工作时，默认会使用`双亲委托模式`。即在类加载的时候，系统会判断当前类是否已经被加载，如果被加载,就会直接返回可用的类，否则就会尝试加载；在尝试加载时，会先请求双亲处理，如果双亲请求失败，则会自己加载。

使用双亲委派模型可以避免重复加载，当父类已经加载了该类的时候，子类不需要再次加载；同时也能避免用户自己编写的类动态替换 Java 的一些核心类。但是在一些系统中需要父类的加载器去请求子类的加载器来加载类，比如 JDBC。JDBC 本身是 Java 连接数据库的一个标准，其接口在启动类加载器中，但是接口的实现由各个数据库厂商来完成，它们在应用类加载器中，这样就会出现接口无法创建由应用类加载器加载的应用实例。

#### JDK 中的 SPI

在 Java 中，把核心类 `rt.jar` 中提供外部服务，可以用应用层自行实现的接口的方式改为 **SPI**。

在 Thread 类中有两个方法：

```java
// 获取线程中的上下文加载器
public ClassLoader getContextClassLoader();
// 设置线程中的上下文加载器
public void setContextClassLoader(ClassLoader cl);
```

通过这个两个方法，可以把一个 ClassLoader 置于一个线程实例中，使该 ClassLoader 成为一个相对共享的实例，即可以实现在启动类加载器中访问由应用类加载器实现的 SPI 接口。

### Dubbo SPI

#### Dubbo SPI 使用

Dubbo 针对 JDK 的 SPI 进行了一些改进，官方文档给出的描述是

> - JDK 标准的 SPI 会一次性实例化拓展点的所有实现，如果有拓展实现初始化很耗时间，但如果没用上也加载，会很浪费资源
> - 如果拓展点加载失败，连拓展点名称都拿不到了
> - 增加了对拓展点 Ioc 和 AOP 的支持，一个拓展点可以直接 setter 注入其他拓展点

概括来说就是**提升性能**和**增加功能**

Dubbo 的 SPI 的相关逻辑被封装在 ExtensionLoader 类中，通过 ExtensionLoader，我们可以加载指定的实现类，Dubbo SPI 是通过键值对的方式进行配置。因此需要将配置文件放在 `META-INF/dubbo` 路径下，如下配置：

```java
optimusPrime = org.apache.spi.OptimusPrime
bumblebee = org.apache.spi.Bumblebee
```

接口类需要使用 `@SPI` 注解，实现类不变

```java
@SPI
public interface Robot {
    void sayHello();
}
```

测试类

```java
@Test
public void sayHello() throws Exception {
    ExtensionLoader<Robot> extensionLoader = ExtensionLoader.getExtensionLoader(Robot.class);
    Robot optimusPrime = extendionLoader.getExtension("optimusPrime");
    optimusPrime.satHello();
    Robot bumblebee = extensionLoader.getExtension("bumblebee");
    bumblebe.sayHello();
}
```

#### SPI 源码解析

在 Dubbo SPI 的使用中，首先通过 ExtensionLoader 的 getExtensionLoader 方法获取一个 ExtensionLoader 实例，然后再通过 ExtensionLoader 的 getExtension 方法获取拓展类对象。

1. getExtensionLoader 方法用于从缓存中获取与拓展类对应的 ExtensionLoader，如果缓存未命中，则创建一个新实例

2. getExtension 首先检查缓存，若缓存未命中则创建拓展对象，源码如下

   ```java
   public T getExtension(String name) {
       if (name == null || name.length() == 0)
           throw new IllegalArgumentException("Extension name == null");
       if ("true".equals(name)) {
           // 获取默认的拓展实现类
           return getDefaultExtension();
       }
       // Holder，顾名思义，用于持有目标对象
       Holder<Object> holder = cachedInstances.get(name);
       if (holder == null) {
           cachedInstances.putIfAbsent(name, new Holder<Object>());
           holder = cachedInstances.get(name);
       }
       Object instance = holder.get();
       // 双重检查
       if (instance == null) {
           synchronized (holder) {
               instance = holder.get();
               if (instance == null) {
                   // 创建拓展实例
                   instance = createExtension(name);
                   // 设置实例到 holder 中
                   holder.set(instance);
               }
           }
       }
       return (T) instance;
   }
   ```

##### createExtension 方法解析

createExtension 方法的流程为：

1. 通过 getExtensionClasses 获取所有的拓展类
2. 通过反射创建拓展对象
3. 向拓展对象中注入依赖
4. 将拓展对象包裹在相应的 Wrapper 对象中

```java
private T createExtension(String name) {
    // 从配置文件中加载所有的拓展类，可得到“配置项名称”到“配置类”的映射关系表
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 通过反射创建实例
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 向实例中注入依赖
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            // 循环创建 Wrapper 实例
            for (Class<?> wrapperClass : wrapperClasses) {
                // 将当前 instance 作为参数传给 Wrapper 的构造方法，并通过反射创建 Wrapper 实例。
                // 然后向 Wrapper 实例中注入依赖，最后将 Wrapper 实例再次赋值给 instance 变量
                instance = injectExtension(
                    (T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("...");
    }
}
```

获取所有的拓展类是加载拓展类的关键，下面也重点解析相关源码

在通过名称获取拓展类之前，首先需要根据配置文件解析出拓展项名称到拓展类的映射关系表 `Map<名称，拓展类>`，之后再根据拓展项名称从映射关系表中取得相应的拓展类，源码如下

```java
private Map<String, Class<?>> getExtensionClasses() {
    // 从缓存中获取已加载的拓展类
    Map<String, Class<?>> classes = cachedClasses.get();
    // 双重检查
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 加载拓展类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

其主要逻辑也是使用双重检查锁机制通过 loadExtensionClasses 加载拓展类。在 loadExtensionClasses 中做了两件事情：一是对 SPI 注解进行解析；二是读取配置文件，通过反射加载类。这部分源码比较多，可以通过引用链接查看原始文章中的代码

#### Dubbo IOC

Dubbo IOC 是通过 setter 方法注入依赖的。Dubbo 首先会通过反射获取到实例的所有方法，然后再遍历方法列表，检测方法名是否具有 setter 方法特性。若有，则通过 ObjectFactory 获取依赖对象，最后通过反射调用 setter 方法将依赖设置到目标对象中，具体源码如下

```java
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            // 遍历目标类的所有方法
            for (Method method : instance.getClass().getMethods()) {
                // 检测方法是否以 set 开头，且方法仅有一个参数，且方法访问级别为 public
                if (method.getName().startsWith("set")
                    && method.getParameterTypes().length == 1
                    && Modifier.isPublic(method.getModifiers())) {
                    // 获取 setter 方法参数类型
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        // 获取属性名，比如 setName 方法对应属性名 name
                        String property = method.getName().length() > 3 ? 
                            method.getName().substring(3, 4).toLowerCase() + 
                            	method.getName().substring(4) : "";
                        // 从 ObjectFactory 中获取依赖对象
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            // 通过反射调用 setter 方法设置依赖
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method...");
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

#### SPI 自适应扩展机制 

在 Dubbo 中，很多拓展都是通过 SPI 机制加载的，比如 Protocol，Cluster，LoadBalance 等，但是有时候，有些拓展并不想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行参数进行加载。笔者自己的理解是实现类似于 Spring AOP 的机制，实现方式也类似。首先 Dubbo 会为拓展接口生成具有代理功能的代码，然后通过 javassist 或者 jdk 编译这段代码，得到 Class 类，最后通过反射创建代理类。

##### 自适应拓展机制简单实现

定义一个车轮制造厂的接口

```java
public interface WheelMaker {
    Wheel makeWheel(URL url);
}
```

WheelMaker 接口的自适应实现类

```java
public class AdaptiveWheelMaker implements WheelMaker {
    public Wheel makeWheel(URL url) {
        if (url == null)
            throw new IllegalArugumentException("url == null");
        // 1. 从 URL 中获取 WheelMaker 名称
        String wheelMakerName = url.getParameter("Wheel.maker");
        if (wheelMakerName == null)
            throw new IllegalArugumentException("wheelMakerName == null");
        // 2. 通过 SPI 加载具体的 WheelMaker
        WheelMaker wheelMaker = ExtensionLoader.getExtensionLoader(WheelMaker.class).getExtension(wheelMakerName);
        // 3. 调用目标方法
        return wheelMaker.makeWheel(URL url);
    }
}
```

CarMaker 接口与实现类

```java
public interface CarMaker {
    Car makeCar(URL url);
}
public class RaceCarMaker implements CarMaker {
    WheelMaker wheelMaker;
    // 通过 setter 注入 AdaptiveWheelMaker
    public setWheelMaker(WheelMaker wheelMaker) {
        this.wheelMaker = wheelMaker;
    }
    public Car makeCar(URL url) {
        Wheel wheel = wheelMaker.makeWheel(url);
        return new RaceCar(wheel, ...);
    }
}
```

简单解释下上面的程序：假设在程序运行时，有一个 url 参数 `dubbo://192.168.0.101:20880/XxxService?wheel.maker=MichelinWheelMaker`，RaceCarMaker 的 makeCar 方法将上面的 url 作为参数传给 AdaptiveWheelMaker 的 makeWheel 方法，makeWheel 方法从 url 中提取 wheel.maker 参数，得到 MichelinWheelMaker。之后再通过 SPI 加载配置名为 MichelinWheelMaker 的实现类，得到具体的 WheelMaker 实例。

##### 源码解析

getAdaptiveExtension 方法是获取自适应拓展的入口方法，相关代码如下：

```java
public T getAdaptiveExtension() {
    // 从缓存中获取自适应拓展
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {    // 缓存未命中
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        // 创建自适应拓展
                        instance = createAdaptiveExtension();
                        // 设置自适应拓展到缓存中
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new 
                            IllegalStateException("fail to create adaptive instance: ...");
                    }
                }
            }
        } else {
            throw new IllegalStateException("fail to create adaptive instance:  ...");
        }
    }

    return (T) instance;
}
```

getAdaptiveExtension 方法逻辑比较简单，缓存未命中时，使用双重检查锁机制调用 createAdaptiveExtension 方法创建自适应拓展。跟进去看到 createAdaptiveExtension  方法相关源码如下：

```java
private T createAdaptiveExtension() {
    try {
        // 获取自适应拓展类，并通过反射实例化
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extension ...");
    }
}
```

createAdaptiveExtension  方法主要有三个逻辑：

1. 调用 getAdaptiveExtensionClass 方法获取自适应拓展 Class 对象
2. 通过反射进行实例化
3. 调用 injectExtension 方法向拓展实例中注入依赖

反射进行实例化比较好理解，注入依赖就是前文所提到的 IoC，主要分析下 getAdaptiveExtensionClass 方法的逻辑，代码如下：

```java
private Class<?> getAdaptiveExtensionClass() {
    // 通过 SPI 获取所有的拓展类
    getExtensionClasses();
    // 检查缓存，若缓存不为空，则直接返回缓存
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    // 创建自适应拓展类
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

getExtensionClasses 方法用于获取某个接口的所有实现类，比如该方法获取 Protocol 接口的 DubboProtocol、HttpProtocol、InjvmProtocol 等实现类。在获取实现类的过程中，如果某个实现类被 Adaptive 注解修饰了，那么该类就会被赋值给 cacheedAdaptiveClass 变量，此时直接返回 cacheedAdaptiveClass 即可。createAdaptiveExtensionClass 方法用于生成自适应拓展类，该方法首先会生成自适应拓展类的源码，然后通过 Compiler 实例（Dubbo 默认使用 javassist 作为编译器）编译源码，得到代理类 Class 实例。