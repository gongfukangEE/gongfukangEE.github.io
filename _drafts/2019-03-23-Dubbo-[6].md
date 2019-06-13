---
layout: post
title:  "Dubbo 服务暴露和引用总结"
categories: [分布式, Dubbo, RPC]
tags:  分布式 Dubbo RPC 
author: G.Fukang
---
Dubbo 服务暴露和引用总结

本文是对前面关于服务暴露和引用的总结与补充，完整 Dubbo 相关博客可在引用中查看

> - [浅析 Dubbo 服务暴露](<https://gongfukangee.github.io/2019/03/18/Dubbo-2/>)
> - [浅析 Dubbo 服务引用机制](<https://gongfukangee.github.io/2019/03/20/Dubbo-3/>)
> - [浅析 Dubbo SPI 机制](<https://gongfukangee.github.io/2019/03/13/Dubbo-1/>)
> - [浅析 Dubbo 服务目录 Directory](<https://gongfukangee.github.io/2019/03/21/Dubbo-4/>)
> - [浅析 Dubbo 集群容错、负载均衡](<https://gongfukangee.github.io/2019/03/23/Dubbo-5/>)

# Dubbo 调用流程

![](http://www.iocoder.cn/images/Dubbo/2018_01_01/01.png)

1. Provider

   0. start 启动服务
   1. register 注册服务到服务中心

2. Consumer

   1. subscribe 向注册中心订阅服务。只订阅使用到的服务，首次会拉取订阅的服务列表，缓存在本地

   2. [异步] notify 当服务发生变化，获取最新的服务列表，更新本地缓存

3. invoke 调用

   1. Conusmer 直接发起对 Provider 的调用，无需注册中心。而对多个 Provider 的负载均衡，Consumer 通过 cluster 组件实现

4. count 监控

   1. [异步] Conusmer 和 Provider 都异步通知监控中心

![](<https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/Dubbo%20%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E5%9B%BE.png>)

# 服务暴露和服务引用

## 解析服务

1. 基于 `dubbo.jar` 内的 `META-INF/spring.handles` 配置，Spring 在遇到 dubbo 名称空间时，会回调 DubboNamespaceHandler
2. 所有的 dubbo 标签，都会统一用 DubboBeanDefinitionParser 进行解析，将 XML 标签解析成 Bean 对象
3. 在 ServiceConfig#export() 或 ReferenceConfig#get() 初始化时，将 Bean 对象转换成 URL 格式，所有 Bean 属性转成 URL 参数
4. 将 URL 传给 Dubbo SPI，基于 SPI 自适应机制，根据 URL 协议投，进行不同服务的暴露或引用。

### 服务暴露

>  官方文[开发者指南 - 实现细节](http://dubbo.apache.org/zh-cn/docs/dev/implementation.html)档给出服务提供者暴露一个服务的详细过程：

![](http://dubbo.apache.org/docs/zh-cn/dev/sources/images/dubbo_rpc_export.jpg)

#### 本地暴露

1. 在没有注册中心，直接暴露提供者的情况下，ServiceConfig 解析出的 URL 格式为：`dubbo://service-host/{服务名}/{版本号}`

2. 基于 Dubbo SPI 的自适应机制，通过 URL  `dubbo://` 协议头识别，直接调用 DubboProtocol#export() 方法，打开服务端口

   1. 根据解析出的 URL，创建一个新的 URL，将协议头设置为 `injvm`，并设置主机名以及端口值

   2. 将具体的服务类名，比如 `DubboServiceInjvmImpl`，通过 ProxyFactory 包装成 Invoker 实例

   3. 使用 InjvmProtocol 暴露 Invoker 实例并将其转化为 Exporter 实例

   4. 最后将生成的 Exporter 实例存放到 ServiceConfig 的 `List<Exporter> exporters` 中

#### 远程暴露

1. 在有注册中心，需要注册提供者地址的情况下，ServiceConfig 解析出的 URL 格式为：`registry:// registry-host/org.apache.dubbo.registry.RegistryService?export=URL.encode("dubbo://service-host/{服务名}/{版本号}")`
2. 基于 Dubbo SPI 的自适应机制，通过 URL `registry://` 协议头识别，就调用 RegistryProtocol#export() 方法
   1. 将具体的服务类名，比如 `DubboServiceRegistryImpl`，通过 ProxyFactory 包装成 Invoker 实例
   2. 调用 doLocalExport 方法，使用 DubboProtocol 将 Invoker 转化为 Exporter 实例，并打开 Netty 服务端监听客户请求
   3. 创建 Registry 实例，连接 Zookeeper，并在服务节点下写入提供者的 URL 地址，注册服务
   4. 向注册中心订阅 override 数据，并返回一个 Exporter 实例
3. 根据 URL 格式中的 `"dubbo://service-host/{服务名}/{版本号}"`中协议头 `dubbo://` 识别，调用 DubboProtocol#export() 方法，开发服务端口
4. RegistryProtocol#export() 返回的 Exporter 实例存放到 ServiceConfig 的 `List<Exporter> exporters` 中 

### 服务引用

> 官方文[开发者指南 - 实现细节](http://dubbo.apache.org/zh-cn/docs/dev/implementation.html)档给出服务提供者暴露一个服务的详细过程：

![](http://dubbo.apache.org/docs/zh-cn/dev/sources/images/dubbo_rpc_refer.jpg)

#### 本地引用

1. 在没有注册中心，直连提供者的情况下，ReferenceConfig 解析出的 URL 格式为：`dubbo://service-host/com.foo.FooService?version=1.0.0`
2. 基于拓展点自适应机制，通过 URL 的 `dubbo://` 协议头识别，直接调用 DubboProtocol 的 refer 方法，直接生成 Invoker
3. Invoker 创建完毕后，调用 ProxyFactory 为服务接口生成代理对象，返回提供者引用

#### 远程引用

1. 从注册中心发现引用服务：在有注册中心，通过注册中心发现提供者地址的情况下，ReferenceConfig 解析出的 URL 格式为：`registry://registry-host:/org.apache.registry.RegistryService?refer=URL.encode("conumer-host/com.foo.FooService?version=1.0.0")`。
2. 通过 URL 的 `registry://` 协议头识别，就会调用 `RegistryProtocol#refer()` 方法
   1. 查询提供者 URL，如 `dubbo://service-host/com.foo.FooService?version=1.0.0` ，来获取注册中心
   2. 创建一个 RegistryDirectory 实例并设置注册中心和协议
   3. 生成 conusmer 连接，在 consumer 目录下创建节点，向注册中心注册
   4. 注册完毕后，订阅 providers，configurators，routers 等节点的数据
   5. 通过 URL 的 `dubbo://` 协议头识别，调用 `DubboProtocol#refer()` 方法，创建一个 ExchangeClient 客户端并返回 DubboInvoker 实例
3. 由于一个服务可能会部署在多台服务器上，这样就会在 providers 产生多个节点，这样也就会得到多个 DubboInvoker 实例，就需要 RegistryProtocol 调用 Cluster 将多个服务提供者节点伪装成一个节点，并返回一个 Invoker
4. Invoker 创建完毕后，调用 ProxyFactory 为服务接口生成代理对象，返回提供者引用

