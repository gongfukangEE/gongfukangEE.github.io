---
layout: post
title:  "浅析 Dubbo 服务引用机制"
categories: [分布式, Dubbo, RPC]
tags:  分布式 Dubbo RPC 
author: G.Fukang
---
浅析 Dubbo 服务引用机制

> 转载自 [Dubbo -- 源码解析 -- 服务引用](http://dubbo.apache.org/zh-cn/docs/source_code_guide/refer-service.html) 笔者对其进行重新排版，并加入自己的理解

# 服务引用方式

1. 直连引用服务：在没有注册中心，直连提供者的情况下，ReferenceConfig 解析出的 URL 格式为：`dubbo://service-host/com.foo.FooService?version=1.0.0`，基于拓展点自适应机制，通过 URL 的 `dubbo://` 协议头识别，直接调用 DubboProtocol 的 refer 方法，返回调用者引用
2. 从注册中心发现引用服务：在有注册中心，通过注册中心发现提供者地址的情况下，ReferenceConfig 解析出的 URL 格式为：`registry://registry-host:/org.apache.registry.RegistryService?refer=URL.encode("conumer-host/com.foo.FooService?version=1.0.0")`。基于拓展点自适应机制，通过 URL 的 `registry://` 协议头识别，就会调用 `RegistryProtocol#refer()` 方法，基于 refer 参数中的条件，查询提供者 URL，如 `dubbo://service-host/com.foo.FooService?version=1.0.0` 。然后基于拓展点自适应机制，通过 URL 的 `dubbo://` 协议头识别，就会调用 `DubboProtocol#refer()` 方法，得到提供者引用。最后 RegistryProtocol 将多个提供者引用，通过 Cluster 拓展点，伪装成单个提供者引用返回

# 服务引用时机

1. 饿汉式：在 Spring 容器调用 ReferenceBean 的 afterPropertiesSet 方法时引用服务
2. 懒汉式[默认]：在 ReferenceBean 对应的服务被注入到其他类中引用

# 源码分析

服务引用的入口方法为 ReferenceBean 的 getObject 方法，该方法定义在 Spring 的 FactoryBean 接口中，ReferenceBean 实现了这个方法，实现代码如下：

```java
public Object getObject() throws Exception {
    return get();
}

public synchronized T get() {
    if (destroyed) {
        throw new IllegalStateException("Already destroyed!");
    }
    // 检测 ref 是否为空，为空则通过 init 方法创建
    if (ref == null) {
        // init 方法主要用于处理配置，以及调用 createProxy 生成代理类
        init();
    }
    return ref;
```

这段代码的逻辑比较简单，如果引用服务为空，则调用 init() 方法创建，其中 init 方法主要用于处理配置和调用 createProxy 生成代理类

## 处理配置

Dubbo 在引用服务时，首先会对配置进行校验，来保证配置的正确性。

继续跟进 init 方法，这块代码比较多，就不贴了，笔者这里简单总结下逻辑：

1. 检测 ConsumerConfig 实例是否存在，如果不存在则创建一个新的实例，然后通过系统变量或者 `dubbo.properties` 配置文件填充 ConsumerConfig 的字段
2. 从系统属性或配置文件中加载与接口名相对应的配置，并将解析结果赋值给 url 字段
3. 检测 consumer，module，application 等核心配置类是否为空，为空则尝试从其他配置类中获取
4. 将 ApplicationConfig，ConsumerConfig，ReferenceConfig 等对象的字段信息添加到 map 中
5. 处理 MethodConfig 实例

## 引用服务

引用服务从 createProxy 开始，它在 init 方法的代码如下

```java
private void init() {
    //${省略部分代码}
    // 配置校验
    //${省略部分代码}
    // 获取服务消费者 ip 地址
    String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
    if (hostToRegistry == null || hostToRegistry.length() == 0) {
        hostToRegistry = NetUtils.getLocalHost();
    } else if (isInvalidLocalHost(hostToRegistry)) {
        throw new IllegalArgumentException("Specified invalid registry ip from property..." );
    }
    map.put(Constants.REGISTER_IP_KEY, hostToRegistry);

    // 存储 attributes 到系统上下文中
    StaticContext.getSystemContext().putAll(attributes);

    // 创建代理类
    ref = createProxy(map);

    // 根据服务名，ReferenceConfig，代理类构建 ConsumerModel，
    // 并将 ConsumerModel 存入到 ApplicationModel 中
    ConsumerModel consumerModel = new ConsumerModel(getUniqueServiceName(), this, ref, interfaceClass.getMethods());
    ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
}
```

跟进 createProxy 方法，代码如下：

```java
private T createProxy(Map<String, String> map) {
    URL tmpUrl = new URL("temp", "localhost", 0, map);
    final boolean isJvmRefer;
    if (isInjvm() == null) {
        // url 配置被指定，则不做本地引用
        if (url != null && url.length() > 0) {
            isJvmRefer = false;
        // 根据 url 的协议、scope 以及 injvm 等参数检测是否需要本地引用
        // 比如如果用户显式配置了 scope=local，此时 isInjvmRefer 返回 true
        } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
            isJvmRefer = true;
        } else {
            isJvmRefer = false;
        }
    } else {
        // 获取 injvm 配置值
        isJvmRefer = isInjvm().booleanValue();
    }

    // 本地引用
    if (isJvmRefer) {
        // 生成本地引用 URL，协议为 injvm
        URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
        // 调用 refer 方法构建 InjvmInvoker 实例
        invoker = refprotocol.refer(interfaceClass, url);
        
    // 远程引用
    } else {
        // 定义直连地址，可以是服务提供者的地址，也可以是注册中心的地址
        if (url != null && url.length() > 0) {
            // 当需要配置多个 url 时，可用分号进行分割，这里会进行切分
            String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                    // 创建 URL 对象
                    URL url = URL.valueOf(u);
                    if (url.getPath() == null || url.getPath().length() == 0) {
                        // 设置接口全限定名为 url 路径
                        url = url.setPath(interfaceName);
                    }
                    
                    // 检测 url 协议是否为 registry，若是，表明用户想使用指定的注册中心
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        // 将 map 转换为查询字符串，并作为 refer 参数的值添加到 url 中
                        urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    } else {
                        // 合并 url，移除服务提供者的一些配置（这些配置来源于用户配置的 url 属性），
                        // 比如线程池相关配置。并保留服务提供者的部分配置，比如版本，group，时间戳等
                        // 最后将合并后的配置设置为 url 查询字符串中。
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else {
            // 加载注册中心 url
            List<URL> us = loadRegistries(false);
            if (us != null && !us.isEmpty()) {
                for (URL u : us) {
                    // 加载监控中心
                    URL monitorUrl = loadMonitor(u);
                    if (monitorUrl != null) {
                        map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                    }
                    // 添加 refer 参数到 url 中，并将 url 添加到 urls 中
                    urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                }
            }

            // 未配置注册中心，抛出异常
            if (urls.isEmpty()) {
                throw new IllegalStateException("No such any registry to reference...");
            }
        }

        // 单个注册中心或服务提供者(服务直连，下同)
        if (urls.size() == 1) {
            // 调用 RegistryProtocol 的 refer 构建 Invoker 实例
            invoker = refprotocol.refer(interfaceClass, urls.get(0));
            
        // 多个注册中心或多个服务提供者，或者两者混合
        } else {
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;

            // 获取所有的 Invoker
            for (URL url : urls) {
                // 通过 refprotocol 调用 refer 构建 Invoker，refprotocol 会在运行时
                // 根据 url 协议头加载指定的 Protocol 实例，并调用实例的 refer 方法
                invokers.add(refprotocol.refer(interfaceClass, url));
                if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    registryURL = url;
                }
            }
            if (registryURL != null) {
                // 如果注册中心链接不为空，则将使用 AvailableCluster
                URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                // 创建 StaticDirectory 实例，并由 Cluster 对多个 Invoker 进行合并
                invoker = cluster.join(new StaticDirectory(u, invokers));
            } else {
                invoker = cluster.join(new StaticDirectory(invokers));
            }
        }
    }
    // ${省略代码，启动时检查}

    // 生成代理类
    return (T) proxyFactory.getProxy(invoker);
}
```

简单概括下这段代码的逻辑：

1. 根据配置检查是否为本地调用，如果是，则调用 InjvmProtocol 的 refer 方法生成 InjvmInvoker 实例；如果不是则读取直连配置项，或注册中心 url，并将读取到的 url 存储到 urls 中
2. 如果 URLs 的数量为 1，则直接通过 Protocol 自适应拓展类构建 Invoker 实例接口；如果 URLs 元素数量大于 1，即存在多个注册中或服务直连 URL，此时先根据 URL 工具 Invoker，然后再通过 Cluster 合并多个 Invoker，最后调用 ProxyFactory 生成代理类

### 创建 Invoker

Invoker 是 Dubbo 的核心模型，代表一个可执行体。在服务引用时，Invoker 用于执行远程调用。Invoker 是由 Protocol 实现类构建来的，Protocol 实现类有很多，最常用的是 RegistryProtocol 和 DubboProtocol。

#### 本地引用

本地引用时序图：

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/Dubbo%20%E6%9C%AC%E5%9C%B0%E5%BC%95%E7%94%A8%20%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

本地引用时调用 `DubboProtocol#refer()` 方法创建 Invoker，因此首先看下在部分的源码

```java
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);
    // 创建 DubboInvoker
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

这段代码的逻辑比较简单，这里有个调用 getClients，这个方法用于获取客户端实例，实例类型为 ExchangeClient。ExchangeClient 并不具备通信能力，它需要更底层的客户端实例进行通信，比如 NettyClient 等。接着跟进去 getClients 方法

```java
private ExchangeClient[] getClients(URL url) {
    // 是否共享连接
    boolean service_share_connect = false;
  	// 获取连接数，默认为0，表示未配置
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
    // 如果未配置 connections，则共享连接
    if (connections == 0) {
        service_share_connect = true;
        connections = 1;
    }

    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (service_share_connect) {
            // 获取共享客户端
            clients[i] = getSharedClient(url);
        } else {
            // 初始化新的客户端
            clients[i] = initClient(url);
        }
    }
    return clients;
}
```

这段代码是根据 connections 数量决定是获取共享客户端还是创建新的客户端。创建新的客户端会调用 initClient 方法，这个方法主要就是根据参数创建 NettyClient 对象，然后调用 Netty 提供的 API 构建 Netty 客户端，具体就不再继续跟进去分析。

#### 远程引用

远程引用时序图：

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/Dubbo%20%E8%BF%9C%E7%A8%8B%E5%BC%95%E7%94%A8%20%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

远程引用时，调用 `RegistryProtocol#refer()` 方法用于生成 Invoker，因此回到 createProxy 方法中，跟进代码第 76-78 行的 refer 方法。

```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    // 获取真实的注册中心 URL
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    // 获取注册中心实例
    Registry registry = registryFactory.getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // 将 url 查询字符串转为 Map
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    // 获取 group 配置
    String group = qs.get(Constants.GROUP_KEY);
    if (group != null && group.length() > 0) {
        if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
                || "*".equals(group)) {
            // 通过 SPI 加载 MergeableCluster 实例，并调用 doRefer 继续执行服务引用逻辑
            return doRefer(getMergeableCluster(), registry, type, url);
        }
    }
    
    // 调用 doRefer 继续执行服务引用逻辑
    return doRefer(cluster, registry, type, url);
}
```

上面这段代码逻辑比较简单，总结下就是：

1. 为 URL 设置协议头，根据 URL 参数加载注册中心实例
2. 获取 group 配置
3. 根据 group 配置决定 doRefer 方法第一个参数类型，调用 doRefer 方法

跟进去 doRefer 方法

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    // 创建 RegistryDirectory 实例
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    // 设置注册中心和协议
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    Map<String, String> parameters = new HashMap<String, String> (directory.getUrl().getParameters());
    // 生成服务消费者链接
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);

    // 注册服务消费者，在 consumers 目录下新节点
    if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                Constants.CHECK_KEY, String.valueOf(false)));
    }

    // 订阅 providers、configurators、routers 等节点数据
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
            Constants.PROVIDERS_CATEGORY
                    + "," + Constants.CONFIGURATORS_CATEGORY
                    + "," + Constants.ROUTERS_CATEGORY));

    // 一个注册中心可能有多个服务提供者，因此这里需要将多个服务提供者合并为一个
    Invoker invoker = cluster.join(directory);
    ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
    return invoker;
}
```

简单总结下这段代码的逻辑：

1. 创建一个 RegistryDirectory 实例，设置注册中心和协议
2. 生成消费者连接，在 consumer 目录下创建节点，向注册中心注册
3. 注册完毕后，订阅 providers，cofigurators，routers 等节点的数据
4. 由于一个服务可能会部署在多台服务器上，这样就会在 providers 产生多个节点，这时候需要 Cluster 将多个服务提供者节点合并为一个节点，并返回一个 Invoker

### 创建代理

Invoker 创建完毕后，接下来要做的就是为服务接口生成代理对象，有了代理对象，就可以进行远程调用。现在继续回到 `createProxy` 方法，该方法的最后就是调用 `proxyFactory.getProxy(invoker)` 生成代理，跟进去继续查看源码：

```java
public <T> T getProxy(Invoker<T> invoker) throws RpcException {
    // 调用重载方法
    return getProxy(invoker, false);
}

public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
    Class<?>[] interfaces = null;
    // ${省略代码，获取接口列表，切分接口列表}
    if (interfaces == null) {
        interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
    }

    // 为 http 和 hessian 协议提供泛化调用支持
    if (!invoker.getInterface().equals(GenericService.class) && generic) {
        int len = interfaces.length;
        Class<?>[] temp = interfaces;
        // 创建新的 interfaces 数组
        interfaces = new Class<?>[len + 1];
        System.arraycopy(temp, 0, interfaces, 0, len);
        // 设置 GenericService.class 到数组中
        interfaces[len] = GenericService.class;
    }

    // 调用重载方法
    return getProxy(invoker, interfaces);
}

public abstract <T> T getProxy(Invoker<T> invoker, Class<?>[] types);
```

这段代码主要是获取 interfaces 数组，继续跟进 getProxy 方法。`getProxy(Invoker,Class<?>[])` 这个方法是一个抽象方法，在 JavassistProxyFactory 类中可以看到该方法具体实现代码。

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    // 生成 Proxy 子类（Proxy 是抽象类）。并调用 Proxy 子类的 newInstance 方法创建 Proxy 实例
    return (T) Proxy.getProxy(interfaces)
        .newInstance(new InvokerInvocationHandler(invoker));
}
```

这段代码逻辑比较简单，就是利用传入的接口参数，通过 JDK 动态代理生成 Proxy 实例，可以继续跟进 getProxy 方法查看生成代理类的具体代码，由于笔者这段代码看的云里雾里，就不继续分析了，想进一步了解的，可以查看引用原文分析。到这里生成代理类就完成了整个服务引用逻辑的分析。