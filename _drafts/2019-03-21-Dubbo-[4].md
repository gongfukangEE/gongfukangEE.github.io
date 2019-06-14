---
layout: post
title:  "浅析 Dubbo 服务目录 Directory"
categories: [分布式, Dubbo, RPC]
tags:  分布式 Dubbo RPC 
author: G.Fukang
---
浅析 Dubbo 服务目录 Directory

> 转载自 [Dubbo -- 源码解析 -- 服务字典](http://dubbo.apache.org/zh-cn/docs/source_code_guide/directory.html) 笔者对其进行重新排版，并加入自己的理解

# 服务目录 Directory

服务目录中存储了一些和服务提供者有关的信息，通过服务目录，服务消费者可获取到服务提供者的信息，比如 ip、端口、服务协议等。在一个服务集群中，服务提供者数量并不是一成不变的，如果集群中新增了一台机器，相应地在服务目录中就要新增一条服务提供者记录。服务目录在获取注册中心的服务配置信息后，会为每条配置信息生成一个 Invoker 对象，并把这个 Invoker 对象存储起来，这个 Invoker 才是服务目录最终持有的对象。简单来说，服务目录可以看作是 Invoker 集合，且这个集合中的元素会随着注册中心的变化而进行动态调整。

服务目录目前内置的实现有两个，分别是 StaticDirectory 和 RegistryDirectory。其中 RegistryDirectory 实现了 NotifyListener 接口，当注册中心节点信息发生变化时后，RegistryDirectory 可以通过此接口方法得到变更信息，并根据变更信息动态调整内部 Invoker 列表。

## 源码分析

### AbstractDirectory 

AbstractDirectory 封装了 Invoker 列举流程，是典型的模板模式，跟进去源码可以看到：

```java
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    if (destroyed) {
        throw new RpcException("Directory already destroyed...");
    }
    
    // 调用 doList 方法列举 Invoker，doList 是模板方法，由子类实现
    List<Invoker<T>> invokers = doList(invocation);
    
    // 获取路由 Router 列表
    List<Router> localRouters = this.routers;
    if (localRouters != null && !localRouters.isEmpty()) {
        for (Router router : localRouters) {
            try {
                // 获取 runtime 参数，并根据参数决定是否进行路由
                if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
                    // 进行服务路由
                    invokers = router.route(invokers, getConsumerUrl(), invocation);
                }
            } catch (Throwable t) {
                logger.error("Failed to execute router: ...");
            }
        }
    }
    return invokers;
}

// 模板方法，由子类实现
protected abstract List<Invoker<T>> doList(Invocation invocation) throws RpcException;
```

上面就是 list 方法的源码，这个封装了 Invoker 的列举过程；

1. 调用 doList 获取 Invoker 列表
2. 根据 Router 的 getUrl 返回值为空与否，已经 runtime 参数决定是否服务路由。如果参数为 true，则每次调用服务前都需要进行服务路由。

**服务路由**

服务路由包含一条路由规则，路由规则决定了服务消费者的调用目标，即规定了服务消费者可调用哪些服务提供者。

服务目录在刷新 Invoker 列表的过程中，会通过 Router 进行服务路由，筛选出符合路由规则的服务提供者。Dubbo 目前提供了三种服务路由实现，分别是条件路由、脚本路由和标签路由。其中条件路由是最常用的，因此这里仅对其进行分析。

条件路由规则由两个条件组成，分别用于对服务消费者和提供者进行匹配，比如这样的条件 `host = 10.20.153.10 => host = 10.20.153.11` 该规则表示 IP 为 10.20.153.10 的服务消费者只能调用 IP 为 10.20.153.11 机器上的服务，不可调用其他机器上的服务。

### StaticDirectory

StaticDirectory 即静态目录，它内部存放的 Invoker 是不会变动的，所有它理论上和不可变的 List 功能相似，具体代码如下：

```java
public class StaticDirectory<T> extends AbstractDirectory<T> {

    // Invoker 列表
    private final List<Invoker<T>> invokers;
    
    // 省略构造方法

    @Override
    public Class<T> getInterface() {
        // 获取接口类
        return invokers.get(0).getInterface();
    }
    
    // 检测服务目录是否可用
    @Override
    public boolean isAvailable() {
        if (isDestroyed()) {
            return false;
        }
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                // 只要有一个 Invoker 是可用的，就认为当前目录是可用的
                return true;
            }
        }
        return false;
    }

    @Override
    public void destroy() {
        if (isDestroyed()) {
            return;
        }
        // 调用父类销毁逻辑
        super.destroy();
        // 遍历 Invoker 列表，并执行相应的销毁逻辑
        for (Invoker<T> invoker : invokers) {
            invoker.destroy();
        }
        invokers.clear();
    }

    @Override
    protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {
        // 列举 Inovker，也就是直接返回 invokers 成员变量
        return invokers;
    }
}
```

### RegistryDirectory

RegistryDirectory 是一种动态服务目录，实现了 NotifyListener 接口。当注册中心服务配置发生变化后，RegistryDirectory 可收到与当前服务相关的变化。

主要有三个逻辑：

1. 列举 Invoker
2. 接收服务变更通知
3. 刷新 Invoker 列表

#### 列举 Invoker

Invoker 列举逻辑封装在 doList 方法中，主要逻辑是从 localMathodInvokerMap 中获取到 Invoker 列表，相关代码如下：

```java
public List<Invoker<T>> doList(Invocation invocation) {
    if (forbidden) {
        // 服务提供者关闭或禁用了服务，此时抛出 No provider 异常
        throw new RpcException(RpcException.FORBIDDEN_EXCEPTION,
            "No provider available from registry ...");
    }
    List<Invoker<T>> invokers = null;
    // 获取 Invoker 本地缓存
    Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap;
    if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
        // 获取方法名和参数列表
        String methodName = RpcUtils.getMethodName(invocation);
        Object[] args = RpcUtils.getArguments(invocation);
        // 检测参数列表的第一个参数是否为 String 或 enum 类型
        if (args != null && args.length > 0 && args[0] != null
                && (args[0] instanceof String || args[0].getClass().isEnum())) {
            // 通过 方法名 + 第一个参数名称 查询 Invoker 列表，具体的使用场景暂时没想到
            invokers = localMethodInvokerMap.get(methodName + "." + args[0]);
        }
        if (invokers == null) {
            // 通过方法名获取 Invoker 列表
            invokers = localMethodInvokerMap.get(methodName);
        }
        if (invokers == null) {
            // 通过星号 * 获取 Invoker 列表
            invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
        }
    }

	// 返回 Invoker 列表
    return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
}
```

#### 接收服务变更通知

RegistryDirectory 是一个动态服务目录，会随注册中心配置的变化进行动态调整，因此，RegistryDirectory 实现了 NotifyListener 接口，通过这个接口获取注册中心变更通知，主要逻辑就是 notify 方法首先是根据 url 的 category 参数对  url 进行分类存储，然后通过 toRouters 和 toConfigurations 将 url 列表转成 Router 和 Configurator 列表，最后调用 referInvoker 方法刷新 Invoker 列表。下面可以看下详细的代码

```java
public synchronized void notify(List<URL> urls) {
    // 定义三个集合，分别用于存放服务提供者 url，路由 url，配置器 url
    List<URL> invokerUrls = new ArrayList<URL>();
    List<URL> routerUrls = new ArrayList<URL>();
    List<URL> configuratorUrls = new ArrayList<URL>();
    for (URL url : urls) {
        String protocol = url.getProtocol();
        // 获取 category 参数
        String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
        // 根据 category 参数将 url 分别放到不同的列表中
        if (Constants.ROUTERS_CATEGORY.equals(category)
                || Constants.ROUTE_PROTOCOL.equals(protocol)) {
            // 添加路由器 url
            routerUrls.add(url);
        } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
            // 添加配置器 url
            configuratorUrls.add(url);
        } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
            // 添加服务提供者 url
            invokerUrls.add(url);
        } else {
            // 忽略不支持的 category
            logger.warn("Unsupported category ...");
        }
    }
    if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
        // 将 url 转成 Configurator
        this.configurators = toConfigurators(configuratorUrls);
    }
    if (routerUrls != null && !routerUrls.isEmpty()) {
        // 将 url 转成 Router
        List<Router> routers = toRouters(routerUrls);
        if (routers != null) {
            setRouters(routers);
        }
    }
    List<Configurator> localConfigurators = this.configurators;
    this.overrideDirectoryUrl = directoryUrl;
    if (localConfigurators != null && !localConfigurators.isEmpty()) {
        for (Configurator configurator : localConfigurators) {
            // 配置 overrideDirectoryUrl
            this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
        }
    }

    // 刷新 Invoker 列表
    refreshInvoker(invokerUrls);
}
```

#### 刷新 Invoker 列表

refreshInvoker 方法是保证 RegistryDirectory 随注册中心变化而变化的关键所在，主要逻辑就是 

1. refreshInvoker 方法首先会根据入参 InvokerUrls 的数量和协议头判断是否禁用所有的服务，如果禁用，则将标志位 forbidden 设为 true，并销毁所有的 Invoker。
2. 如果不禁用，则将 url 转为 Invoker，得到 `<url, Invoker>` 的映射关系，然后进一步转换，得到 `<methodName, Invoker 列表>` 映射关系，之后进行多组 Invoker 合并操作，将合并结果赋值给 methodInvokerMap。methodInvokerMap 变量在 doList 方法中会被用到，用于 Invoker 的列举。
3. 当新的 Invoker 列表生成后，还需要销毁无用的 Invoker，避免服务消费者调用已经下线的服务

```java
private void refreshInvoker(List<URL> invokerUrls) {
    // invokerUrls 仅有一个元素，且 url 协议头为 empty，此时表示禁用所有服务
    if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
            && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
        // 设置 forbidden 为 true
        this.forbidden = true;
        this.methodInvokerMap = null;
        // 销毁所有 Invoker
        destroyAllInvokers();
    } else {
        this.forbidden = false;
        Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap;
        if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
            // 添加缓存 url 到 invokerUrls 中
            invokerUrls.addAll(this.cachedInvokerUrls);
        } else {
            this.cachedInvokerUrls = new HashSet<URL>();
            // 缓存 invokerUrls
            this.cachedInvokerUrls.addAll(invokerUrls);
        }
        if (invokerUrls.isEmpty()) {
            return;
        }
        // 将 url 转成 Invoker
        Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);
        // 将 newUrlInvokerMap 转成方法名到 Invoker 列表的映射
        Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap);
        // 转换出错，直接打印异常，并返回
        if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
            logger.error(new IllegalStateException("urls to invokers error ..."));
            return;
        }
        // 合并多个组的 Invoker
        this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
        this.urlInvokerMap = newUrlInvokerMap;
        try {
            // 销毁无用 Invoker
            destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap);
        } catch (Exception e) {
            logger.warn("destroyUnusedInvokers error. ", e);
        }
    }
}
```
