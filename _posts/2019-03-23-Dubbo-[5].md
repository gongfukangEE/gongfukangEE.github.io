---
layout: post
title:  "浅析 Dubbo 集群容错、负载均衡"
categories: [分布式, Dubbo, RPC]
tags:  分布式 Dubbo RPC 
author: G.Fukang
---
浅析 Dubbo 集群容错、负载均衡

> 转载自 [Dubbo -- 源码解析 -- 集群](http://dubbo.apache.org/zh-cn/docs/source_code_guide/cluster.html) 笔者对其进行重新排版，并加入自己的理解

# 集群

为了避免的单点故障，应用一般部署在至少两台服务器上，也就是说同一环境下的服务提供者数量会大于 1。对于服务消费者来说，同一环境下出现了多个服务提供者。这时会出现一个问题，服务消费者需要决定选择哪个服务提供者进行调用。另外服务调用失败时的处理措施也是需要考虑的，是重试呢，还是抛出异常，亦或是只打印异常等。为了处理这些问题，Dubbo 定义了集群接口 Cluster 以及 Cluster Invoker。集群 Cluster 用途是将多个服务提供者合并为一个 Cluster Invoker，并将这个 Invoker 暴露给服务消费者。这样一来，服务消费者只需通过这个 Invoker 进行远程调用即可，至于具体调用哪个服务提供者，以及调用失败后如何处理等问题，现在都交给集群模块去处理。集群模块是服务提供者和服务消费者的中间层，为服务消费者屏蔽了服务提供者的情况，这样服务消费者就可以专心处理远程调用相关事宜

Dubbo 中的集群容错组件有：Cluster、Cluster Invoker、Directory、Router 和 LoadBalance 等。

![](http://dubbo.apache.org/docs/zh-cn/source_code_guide/sources/images/cluster.jpg)

集群工作过程分为两个阶段：

1. 服务消费者初始化期间，集群 Cluster 实现类为服务消费者创建 Cluster Invoker 实例，即 merge 操作
2. 服务消费者进行远程调用时。以 FailoverClusterInvoker 为例，该类型 Cluster Invoker 首先会调用 Directory 的 list 方法列举 Invoker 列表（可以将 Invoker 简单理解为服务提供者）。Directory 的用途是保存 Invoker，可简单类比为 `List<Invoker>`，其实现类 RegistryDirectory 是一个动态服务目录，可感知注册中心配置的变化，它所持有的 Invoker 列表会随着注册中心内容的变而变化，每次变化后，RegistryDirectory 会动态增删 Invoker，并调用 Router 的 route 方法进行路由，过滤掉不符合路由规则的 Invoker。当 FailoverClusterInvoker 拿到 Directory 返回的 Invoker 列表后，它会通过 LoadBalance 从 Invoker 列表中选择一个 Invoker。最后 FailoverClusterInvoker 会将参数传给 LoadBalance 选择出的 Invoker 实例的 Invoker 方法，进行真正的调用

### 源码分析

在前面可以看到有两个概念，集群接口 Cluster 和 Cluster Invoker。Cluster Invoker 是一种 Invoker，服务提供者的选择逻辑，以及远程调用失败的处理逻辑均是封装在 Cluster Invoker 中。Cluster 是接口，用于生成 Cluster Invoker

### Cluster 实现类分析

Cluster 的实现类有 FailoverCluster、FailfastCluster、FailsafeCluster、FailbackCluster、ForkingCluster 和 BroadcastClusterInvoker 几种，其实现也是基本一样，以 FailoverCluster 为例，来看下源码实现

```java
public class FailoverCluster implements Cluster {
    public final static String NAME = "failover";
    
    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        // 创建并返回 FailoverClusterInvoker 对象
        return new FailoverClusterInvoker<T>(directory);
    }
}
```

### Cluster Invoker 分析

Cluster Invoker 父类为 AbstractClusterInvoker ，在服务消费者进行远程调用时，AbstractClusterInvoker 的 invoker 方法会被调用。列举 Invoker，负载均衡等操作会在此阶段被执行。

首先看下 invoker 方法的逻辑

```java
public Result invoke(final Invocation invocation) throws RpcException {
    checkWhetherDestroyed();
    LoadBalance loadbalance = null;

    // 绑定 attachments 到 invocation 中.
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        ((RpcInvocation) invocation).addAttachments(contextAttachments);
    }

    // 列举 Invoker
    List<Invoker<T>> invokers = list(invocation);
    if (invokers != null && !invokers.isEmpty()) {
        // 加载 LoadBalance
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                .getMethodParameter(RpcUtils.getMethodName(invocation), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
    }
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    
    // 调用 doInvoke 进行后续操作
    return doInvoke(invocation, invokers, loadbalance);
}

// 抽象方法，由子类实现
protected abstract Result doInvoke(Invocation invocation, List<Invoker<T>> invokers,
                                       LoadBalance loadbalance) throws RpcException;
```

AbstractClusterInvoker 的 invoke 方法主要用于列举 Invoker，以及加载 LoadBalance，最后再调用模板方法 doInvoker 进行后续操作。

#### 容错策略

**FailoverClusterInvoker**

FailClusterInvoker 在调用失败时，会自动切换 Invoker 进行重试，**这也是 Dubbo 的默认配置**

**FailbackClusterInvoker**

FailbackClusterInvoker 会在调用失败后，返回一个空结果给调用者，并通过定时任务对失败的调用进行重传，适合执行消息通知等操作

**FailfastCulsterInvoker**

FailfastCulsterInvoker 只会调用一次，失败后立即抛出异常，适合幂等操作，比如新增记录

**FailsafeCulsterInvoker**

FailsafeCulsterInvoker 是一种失败安全的 ClusterInvoker，当调用过程中出现异常时，FailsafeCulsterInvoker 仅会打印异常，而不会抛出异常，适合日志写入

**ForkingClusterInvoker**

ForkingClusterInvoker 会在运行时通过线程池创建多个线程，并发调用多个服务提供者，只有有一个服务提供者成功返回了结果，doInvoker 方法就会立即结束运行

**BroadcastClusterInvoker**

BroadcastClusterInvoker 会逐个调用每个服务提供者，如果其中一台报错，在循环调用结束后，BroadcastClusterInvoker 会抛出异常

# 负载均衡

LoadBalance 中文意思为负载均衡，它的职责是将网络请求，或者其他形式的负载“均摊”到不同的机器上。避免集群中部分服务器压力过大，而另一些服务器比较空闲的情况。通过负载均衡，可以让每台服务器获取到适合自己处理能力的负载。在为高负载服务器分流的同时，还可以避免资源浪费，一举两得。

Dubbo 提供了4种负载均衡实现，分别是基于权重随机算法的 RandomLoadBalance、基于最少活跃调用数算法的 LeastActiveLoadBalance、基于 hash 一致性的 ConsistentHashLoadBalance，以及基于加权轮询算法的 RoundRobinLoadBalance

## 基于权重随机算法的 RandomLoadBalance（默认）

RandomLoadBalance 是加权随机算法的具体实现，它的算法思想很简单。假设我们有一组服务器 servers = [A, B, C]，他们对应的权重为 weights = [5, 3, 2]，权重总和为10。现在把这些权重值平铺在一维坐标值上，[0, 5) 区间属于服务器 A，[5, 8) 区间属于服务器 B，[8, 10) 区间属于服务器 C。接下来通过随机数生成器生成一个范围在 [0, 10) 之间的随机数，然后计算这个随机数会落到哪个区间上。比如数字3会落到服务器 A 对应的区间上，此时返回服务器 A 即可。权重越大的机器，在坐标轴上对应的区间范围就越大，因此随机数生成器生成的数字就会有更大的概率落到此区间内。只要随机数生成器产生的随机数分布性很好，在经过多次选择后，每个服务器被选中的次数比例接近其权重比例。比如，经过一万次选择后，服务器 A 被选中的次数大约为5000次，服务器 B 被选中的次数约为3000次，服务器 C 被选中的次数约为2000次。当然 RandomLoadBalance 缺点也很明显，当调用次数比较少时，Random 产生的随机数可能会比较集中，此时多数请求会落到同一台服务器上

## LeastActiveLoadBalance 最小活跃数负载均衡

LeastActiveLoadBalance 翻译过来是最小活跃数负载均衡。活跃调用数越小，表明该服务提供者效率越高，单位时间内可处理更多的请求。此时应优先将请求分配给该服务提供者。在具体实现中，每个服务提供者对应一个活跃数 active。初始情况下，所有服务提供者活跃数均为0。每收到一个请求，活跃数加1，完成请求后则将活跃数减1。在服务运行一段时间后，性能好的服务提供者处理请求的速度更快，因此活跃数下降的也越快，此时这样的服务提供者能够优先获取到新的服务请求、这就是最小活跃数负载均衡算法的基本思想。

## 基于 Hash 一致性的 ConsistentHashLoadBalance 

首先根据 ip 或者其他的信息为缓存节点生成一个 hash，并将这个 hash 投射到 [0, 232 - 1] 的圆环上。当有查询或写入请求时，则为缓存项的 key 生成一个 hash 值。然后查找第一个大于或等于该 hash 值的缓存节点，并到这个节点中查询或写入缓存项。如果当前节点挂了，则在下一次查询或写入缓存时，为缓存项查找另一个大于其 hash 值的缓存节点即可。大致效果如下图所示，每个缓存节点在圆环上占据一个位置。如果缓存项的 key 的 hash 值小于缓存节点 hash 值，则到该缓存节点中存储或读取缓存项

## 基于加权轮询算法的 RoundRobinLoadBalance

所谓轮询是指将请求轮流分配给每台服务器。举个例子，我们有三台服务器 A、B、C。我们将第一个请求分配给服务器 A，第二个请求分配给服务器 B，第三个请求分配给服务器 C，第四个请求再次分配给服务器 A。这个过程就叫做轮询。轮询是一种无状态负载均衡算法，实现简单，适用于每台服务器性能相近的场景下。但现实情况下，我们并不能保证每台服务器性能均相近。如果我们将等量的请求分配给性能较差的服务器，这显然是不合理的。因此，这个时候我们需要对轮询过程进行加权，以调控每台服务器的负载。经过加权后，每台服务器能够得到的请求数比例，接近或等于他们的权重比。比如服务器 A、B、C 权重比为 5:2:1。那么在8次请求中，服务器 A 将收到其中的5次请求，服务器 B 会收到其中的2次请求，服务器 C 则收到其中的1次请求。

