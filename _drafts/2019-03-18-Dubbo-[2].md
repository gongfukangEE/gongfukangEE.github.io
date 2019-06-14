---
layout: post
title:  "浅析 Dubbo 服务暴露"
categories: [分布式, Dubbo, RPC]
tags:  分布式 Dubbo RPC 
author: G.Fukang
---
浅析 Dubbo 服务暴露机制：本地暴露和远程暴露

>  转载自 [Dubbo -- 源码解析 -- 服务导出](http://dubbo.apache.org/zh-cn/docs/source_code_guide/export-service.html) 笔者对其进行重新排版，并加入自己的理解

# 暴露方式

 Dubbo 的服务暴露有两种方式：

 1. 本地暴露，JVM 本地调用

    在没有注册中心，直接暴露提供者的情况下，ServiceConfig 解析出的 URL 格式为：`dubbo://service-host/com.foo.FooService?version=1.0.0`。基于拓展点自适应机制，通过 URL 的 `dubbo://` 协议头识别，直接调用 DubboProtocol 的 export 方法，打开服务端口

 2. 远程暴露，网络远程调用

    1. 在有注册中心，通过注册中心发现提供者地址的情况下，ReferenceConfig 解析出的 URL 格式为：`registry://registry-host/org.apache.dubbo.registry.RegistryService?refer=URL.encode("consumer://consumer-host/com.foo.FooService?version=1.0.0")`。
    2. 基于拓展点自适应机制，通过 URL 的 `registry://`协议头识别，就会调用 RegistryProtocol 的 refer 参数中的条件，查询提供者 URL，如 `dubbo://service-host/com.foo.FooService?version=1.0.0`。
    3. 基于拓展点自适应机制，通过提供者 URL 的 `dubbo://` 协议头识别，就会调用 DubboProtocol 的 refer 方法，得到提供者引用
    4. 然后 RegistryProtocol 将多个提供者引用，通过 Cluster 拓展点，伪装成单个提供者引用返回

# 服务导出过程

 Dubbo 的服务导出过程始于 Spring 容器发布刷新事件，Dubbo 在接收到事件后，会立即执行服务导出逻辑，整个逻辑流程为：

 1. 前置工作：用于检查参数，组装 URL
 2. 导出服务：本地服务暴露和远程服务暴露
 3. 注册服务：向注册中心注册服务，用于服务发现

## 入口方法

 服务导出入口方法是 ServiceBean 的 onApplicaitonEvent，它是一个事件响应方法，该方法在收到 Spring 上下文刷新事件后执行服务导出操作

 ```java
 public void onApplicationEvent(ContextRefreshedEvent event) {
     // 是否有延迟导出 && 是否已导出 && 是不是已被取消导出
     if (isDelay() && !isExported() && !isUnexported()) {
         // 导出服务
         export();
     }
 }
 ```

## 前置工作

 前置工作主要包括两个部分：校验 ServiceConfig 对象的配置项和组成 Dubbo URL 数组

### 校验配置

 onappliacationEvent 在一系列判断后，开始调用 export 方法

 ```java
 public synchronized void export() {
     if (provider != null) {
         // 获取 export 和 delay 配置
         if (export == null) {
             export = provider.getExport();
         }
         if (delay == null) {
             delay = provider.getDelay();
         }
     }
     // 如果 export 为 false，则不导出服务
     if (export != null && !export) {
         return;
     }
 
     // delay  0，延时导出服务
     if (delay != null && delay  0) {
         delayExportExecutor.schedule(new Runnable() {
             @Override
             public void run() {
                 doExport();
             }
         }, delay, TimeUnit.MILLISECONDS);
         
     // 立即导出服务
     } else {
         doExport();
     }
 }
 ```

 这块的逻辑比较简单，就不做分析，代码继续跟进 `doExport()`，这部分代码比较长，就不贴了，简单总结下逻辑：

 1. 检测 `<udbbo:service` 标签的 interface 属性合法性，不合法则抛出异常
 2. 检测 ProviderConfig 和 ApplicationConfig 等核心配置类对象是否为空，若为空，则尝试从其他配置类对象中获取响应的实例
 3. 检测并处理泛化服务和普通服务类
 4. 检测本地存根配置，并进行相应处理
 5. 对 Application 和 RegistryConfig 等配置类进行检测，为空则尝试创建，若无法创建则抛出异常

### 组装 URL

 URL 是 Dubbo 配置的载体，通过 URL 可以让 Dubbo 的各种配置在各个模块之间传递。组装 URL 就是将一些信息，比如版本、时间戳、方法名以及各种配置对象的字段信息放入到 map 中，map 中的内容将作为 URL 的查询字符串。构建好 map 后，紧接着获取上下文路径、主机名以及端口号等信息，最后将 map 和主机名等数据传给 URL 构造方法创建 URL 对象。

## 导出 Dubbo 服务

 首先从宏观上看下服务导出的逻辑

 ```java
 private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL registryURLs) {
     
     // 省略无关代码
     
     if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
             .hasExtension(url.getProtocol())) {
         // 加载 ConfiguratorFactory，并生成 Configurator 实例，然后通过实例配置 url
         url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                 .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
     }
 
     String scope = url.getParameter(Constants.SCOPE_KEY);
     // 如果 scope = none，则什么都不做
     if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
         // scope != remote，导出到本地
         if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
             exportLocal(url);
         }
 
         // scope != local，导出到远程
         if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
             if (registryURLs != null && !registryURLs.isEmpty()) {
                 for (URL registryURL : registryURLs) {
                     url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                     // 加载监视器链接
                     URL monitorUrl = loadMonitor(registryURL);
                     if (monitorUrl != null) {
                         // 将监视器链接作为参数添加到 url 中
                         url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                     }
 
                     String proxy = url.getParameter(Constants.PROXY_KEY);
                     if (StringUtils.isNotEmpty(proxy)) {
                         registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                     }
 
                     // 为服务提供类(ref)生成 Invoker
                     Invoker<? invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                     // DelegateProviderMetaDataInvoker 用于持有 Invoker 和 ServiceConfig
                     DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
 
                     // 导出服务，并生成 Exporter
                     Exporter<? exporter = protocol.export(wrapperInvoker);
                     exporters.add(exporter);
                 }
                 
             // 不存在注册中心，仅导出服务
             } else {
                 Invoker<? invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                 DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
 
                 Exporter<? exporter = protocol.export(wrapperInvoker);
                 exporters.add(exporter);
             }
         }
     }
     this.urls.add(url);
 }
 ```

 不管是导出本地还是远程，进行服务导出之前，都需要先创建一个 invoker。关于 invoker 这块，笔者也没有理解清楚，就不做详细探讨，仅引用下官网文档的说明：

  Invoker 是实体域，它是 Dubbo 的核心模型，其他模型都是向它靠拢，或者转换它，它代表一个可执行体，可向它发起 invoke 调用，它可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现

 这块想深入了解可以点进引用原文查看。

### 导出服务到本地

**本都暴露时序图**

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/分布式/Dubbo%20本地暴露%20时序图.png)

 导出服务到本地的相关代码如下：

 ```java
 private void exportLocal(URL url) {
     // 如果 URL 的协议头等于 injvm，说明已经导出到本地了，无需再次导出
     if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
         URL local = URL.valueOf(url.toFullString())
             .setProtocol(Constants.LOCAL_PROTOCOL)    // 设置协议头为 injvm
             .setHost(LOCALHOST)
             .setPort(0);
         ServiceClassHolder.getInstance().pushServiceClass(getServiceClass(ref));
         // 创建 Invoker，并导出服务，这里的 protocol 会在运行时调用 InjvmProtocol 的 export 方法
         Exporter<? exporter = protocol.export(
             proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
         exporters.add(exporter);
     }
 }
 ```

 exportLocal 逻辑比较简单，首先根据 URL 协议头决定是否导出服务（URL 协议头等于 injvm），如果需要导出，则创建一个新的 URL 并将协议头、主机名以及端口设置为新的值，然后创建 Invoker，并调用 InjvmProtocol 的 export 方法导出服务，export 方法仅仅创建了一个 InjvmExport

### 导出服务到远程

**远程暴露时序图**

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/Dubbo%20%E8%BF%9C%E7%A8%8B%E6%9A%B4%E9%9C%B2%20%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

 导出服务到远程主要包含服务导出和服务注册两个过程。

#### 服务导出

 服务导出的代码在 RegistryProtocol 的 export 方法上

 ```java
 public <T Exporter<T export(final Invoker<T originInvoker) throws RpcException {
     // 导出服务
     final ExporterChangeableWrapper<T exporter = doLocalExport(originInvoker);
 
     // 获取注册中心 URL
     URL registryUrl = getRegistryUrl(originInvoker);
 
     // 根据 URL 加载 Registry 实现类，比如 ZookeeperRegistry
     final Registry registry = getRegistry(originInvoker);
     
     // 获取已注册的服务提供者 URL
     final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);
 
     // 获取 register 参数
     boolean register = registeredProviderUrl.getParameter("register", true);
 
     // 向服务提供者与消费者注册表中注册服务提供者
     ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);
 
     // 根据 register 的值决定是否注册服务
     if (register) {
         // 向注册中心注册服务
         register(registryUrl, registeredProviderUrl);
         ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
     }
 
     // 获取订阅 URL，比如：
     final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
     // 创建监听器
     final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
     overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
     // 向注册中心进行订阅 override 数据
     registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
     // 创建并返回 DestroyableExporter
     return new DestroyableExporter<T(exporter, originInvoker, overrideSubscribeUrl, registeredProviderUrl);
 }
 ```

 简单总结下这块的业务逻辑：

 1. 调用 doLoacalExport 导出服务
 2. 向注册中心注册服务
 3. 向注册中心订阅 override 数据
 4. 创建并返回 DestroyableExporter

 首先分析下 doLocalExport 方法的逻辑：

 ```java
 private <T ExporterChangeableWrapper<T doLocalExport(final Invoker<T originInvoker) {
     String key = getCacheKey(originInvoker);
     // 访问缓存
     ExporterChangeableWrapper<T exporter = (ExporterChangeableWrapper<T) bounds.get(key);
     if (exporter == null) {
         synchronized (bounds) {
             exporter = (ExporterChangeableWrapper<T) bounds.get(key);
             if (exporter == null) {
                 // 创建 Invoker 为委托类对象
                 final Invoker<? invokerDelegete = new InvokerDelegete<T(originInvoker, getProviderUrl(originInvoker));
                 // 调用 protocol 的 export 方法导出服务
                 exporter = new ExporterChangeableWrapper<T((Exporter<T) protocol.export(invokerDelegete), originInvoker);
                 
                 // 写缓存
                 bounds.put(key, exporter);
             }
         }
     }
     return exporter;
 }
 ```

 这段代码的逻辑比较简单，就是双重检查锁调用 DubboProtocol 的 export 方法，跟进去 export 方法，相关代码如下：

 ```java
 public <T Exporter<T export(Invoker<T invoker) throws RpcException {
     URL url = invoker.getUrl();
 
     // 获取服务标识:服务组名/服务名/服务版本号/端口
     // demoGroup/com.alibaba.dubbo.demo.DemoService:1.0.1:20880
     String key = serviceKey(url);
     // 创建 DubboExporter
     DubboExporter<T exporter = new DubboExporter<T(invoker, key, exporterMap);
     // 将 <key, exporter 键值对放入缓存中
     exporterMap.put(key, exporter);
 
     // 本地存根相关代码
     Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
     Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
     if (isStubSupportEvent && !isCallbackservice) {
         String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
         if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
             // 省略日志打印代码
         } else {
             stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
         }
     }
 
     // 启动服务器
     openServer(url);
     // 优化序列化
     optimizeSerialization(url);
     return exporter;
 }
 ```

 至此就完成了服务导出的过程，后续启动服务器 Dubbo 使用的是 Netty 实现，笔者对此不熟悉，就没有继续跟进去。想要深入理解，可以点进引用原文查看分析。

#### 服务注册

 服务注册对于 Dubbo 来说不是必须的，通过服务直连的方式可以绕过注册中心，但是这样不利于服务治理。Dubbo 支持多种注册中心，一般使用的是 Zookeeper，下文的分析也是基于 Zookeeper。

服务注册的入口方法在 RegistryProtocol 的 export 方法，代码如下：

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    
    // ${导出服务}
    
    // 省略其他代码
    
    boolean register = registeredProviderUrl.getParameter("register", true);
    if (register) {
        // 注册服务
        register(registryUrl, registeredProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }
    
    // ${订阅 override 数据}

    // 省略部分代码
}
```

前面已经分析了导出服务，接下在主要分析服务注册逻辑，跟进 register 查看代码

```java
public void register(URL registryUrl, URL registedProviderUrl) {
    // 获取 Registry
    Registry registry = registryFactory.getRegistry(registryUrl);
    // 注册服务
    registry.register(registedProviderUrl);
}
```

register 方法包含两步操作，获取注册中心实例和向注册中心注册服务

**获取注册中心实例**

getRegistry 方法跟进去代码如下：

```java
public Registry getRegistry(URL url) {
    url = url.setPath(RegistryService.class.getName())
            .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
            .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
    String key = url.toServiceString();
    LOCK.lock();
    try {
    	// 访问缓存
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
            return registry;
        }
        
        // 缓存未命中，创建 Registry 实例
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry...");
        }
        
        // 写入缓存
        REGISTRIES.put(key, registry);
        return registry;
    } finally {
        LOCK.unlock();
    }
}
```

代码逻辑比较简单，先访问缓存，缓存未命中则调用 createRegistry 创建 Registry，然后写入缓存。createRegistry 是一个模板方法，由子类 ZookeeperRegistryFactory 实现的，主要就是利用 Curator 框架创建 Zookeeper 客户端，创建好 Zookeeper 客户端，注册中心的创建过程就结束了。

**节点创建**

服务注册，本质上就是将服务配置数据写入到 Zookeeper 的某个路径的节点下。

![](http://dubbo.apache.org/docs/zh-cn/user/sources/images/zookeeper.jpg)

流程说明：

- 服务提供者启动时：向 `/dubbo/com.foo.BarService/providers` 目录下写入自己的 URL 地址
- 服务消费者启动时：订阅 `/dubbo/com.foo.BarService/providers` 目录下的提供者 URL 地址。并向 `/dubbo/com.foo.BarService/consumers` 目录下写入自己的 URL 地址
- 监控执行启动时：订阅 `/dubbo/com.foo.BarService` 目录下的所有提供者和消费者的 URL 地址

服务注册的接口为 register(URL) 这个方法定义在 FailbackRegistry 抽象类中

```java
public void register(URL url) {
    // ${参数检验}
}
protected abstract void doRegister(URL url);
```

我们重点关注的是 doRegister 方法调用，它是一个模板方法，在子类 ZookeeperRegistry 中实现

```java
protected void doRegister(URL url) {
    try {
        // 通过 Zookeeper 客户端创建节点，节点路径由 toUrlPath 方法生成
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register...");
    }
}
```

这段代码的逻辑比较简单，主要是调用 Zookeeper 客户端创建服务节点。节点路径格式为：`/${group}/${serviceInterface}/providers/${url}`，比如 `/dubbo/org.apache.dubbo.DemoService/providers/dubbo%3A%2F%2F127.0.0.1......`，创建完节点，整个注册过程就分析完了。