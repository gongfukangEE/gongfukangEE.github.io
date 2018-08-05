---
layout: post
title:  "大型网站技术架构[2] -- 网站的高性能架构"
categories: WEB开发 Java-Web
tags:  Java-Web 架构
author: G.Fukang
---
* content
{:toc}
《大型网站技术架构：核心原理与案例分析》读书笔记

# 索引

[大型网站技术架构[1] -- 大型网站架构概述](https://gongfukangee.github.io/2018/06/27/Web-Site-Technology-Framework-1/)

[大型网站技术架构[2] -- 网站的高性能架构]()

# 1、性能测试

网站性能是客观的指标，可以具体体现到响应时间、吞吐量等技术指标，同时也是一种主管感受。

## 1.1 性能测试指标

### 1.1.1 响应时间

应用执行一个操作需要的时间，包括从发出请求开始到收到最后响应数据所需要的时间

### 1.1.2 并发数

指系统能够同时处理请求的数目，测试程序可以通过多线程模拟并发用户的办法来测试系统的并发能力

### 1.1.3 吞吐量

单位时间内系统处理请求的数量，量化指标包括：**TPS**（每秒事务数）、**HPS**（每秒 HTTP 请求数）、**QPS**（每秒查询数）

在系统并发数由小逐渐增大的过程中，系统吞吐量是逐渐增加，达到一个极限后，随着并发数的增加反而下降，达到系统崩溃点后，系统资源耗尽，吞吐量为零。

### 1.1.4 性能计数器

这是描述服务器或操作系统性能的一些数据指标，主要为 System Load（系统负载），指当前正在被 CPU 执行和等待被 CPU 执行的进程数目总和，是反映系统忙闲程度的重要指标，多核 CPU 的情况下，完美情况是所有 CPU 都在使用，没有进程在等待，所以理想值为 CPU 的数目

## 1.2 性能测试方法

**性能测试、负载测试、压力测试、稳定性测试**

# 2、性能优化

## 2.1 Web 前端性能优化

### 2.1.1 浏览器访问优化

- 减少 http 请求（合并 CSS、合并 JavaScript、合并图片）
- 使用浏览器缓存
- 启用压缩
- CSS 放在页面最上面，JavaScript 放在页面最下面
- 减少 Cookie 传输

### 2.1.2 CDN 加速

CDN（内容分发网络）本质仍然是一个缓存，而且将数据缓存在离用户最近的地方，使用户以最快的速度获取数据

由于 CDN 部署在网络运营商的机房，这些运营商又是终端用户的网络服务提供商，因此用户请求路由的第一跳就到达了 CDN 服务器，当 CDN 中存在浏览器请求的资源的时候，从 CDN 直接返回给浏览器，最短路径返回响应，加快用户访问速度，减少数据中心负载压力。

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/Framework_11.jpg)

### 2.1.3 反向代理

代理服务器也可以通过配置缓存功能加速 Web 请求，当用户第一次访问静态内容的时候，静态内容就被缓存在反向代理服务器上，这样当其他用户访问该静态内容的时候，就直接从反向代理服务器返回，加速 Web 请求响应速度，减轻 Web 服务器压力。

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/Framework_12.jpg)

## 2.2 应用服务器性能优化

### 2.2.1 分布式缓存

**缓存可用性**：缓存是为了提高数据库读取性能的，缓存数据丢失或者缓存不可用不会影响到程序的处理，但是，如果缓存服务崩溃时，数据库会因为完全不能承受如此大的压力而宕机，进而影响这个网站不可用。这时候可以通过缓存热备，将缓存访问切换到热备服务器上来解决问题；如果是分布式缓存服务器集群，某台缓存服务器宕机只会有部分缓存数据丢失，可以从重新数据库中加载这部分数据。

**缓存预热**：缓存中存放的是热点数据，热点数据又是利用 LRU（最近久未使用算法）对不断访问的数据筛选淘汰出来的，这个过程需要耗费很长时间，因此可以在缓存系统启动时就把热点数据加载好，这个手段叫做缓存预热。

**缓存穿透**：如果因为业务不当，持续高并发请求某个不存在的数据，由于缓存没有保存该数据，所有的请求都会落在数据库上，会对数据库造成很大的压力，甚至崩溃。简单的对策是将不存在的数据也缓存起来（其 value 为 null）。

### 2.2.2 异步操作

使用消息队列将调用异步化（消峰）：在不使用消息队列的时候，用户的请求数据直接写入数据库，在高并发的情况下会对数据库造成巨大的压力，同时也使得响应延迟加剧；在使用消息队列后，用户请求的数据发送给消息队列后立即返回，再由消息队列的消费者进程从消息队列获取数据，异步写入数据库。

### 2.2.3 使用集群和代码优化

- 多线程
- 资源复用
- 垃圾回收
