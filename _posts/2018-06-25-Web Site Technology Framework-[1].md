---
layout: post
title:  "大型网站技术架构【1】-- 大型网站架构概述"
categories: WEB开发 Java-Web
tags:  Java-Web 架构
author: G.Fukang
---
* content
{:toc}
《大型网站技术架构：核心原理与案例分析》读书笔记

# 索引

[大型网站技术架构【1】-- 大型网站架构概述]()

# 大型网站架构演化

## 大型网站软件系统特点

- 高并发，大流量
- 高可用
- 海量数据
- 用户分布广泛，网络情况复杂
- 安全环境恶劣
- 需求快速变更，发布频繁
- 渐进式发展

## 大型网站架构演化

### 初始阶段

应用程序，数据库，文件等所有的资源都部署在一台服务器上。

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/Framework_1.jpg)

### 应用服务和数据服务分离

随着越来越多的用户访问，导致性能越来越差，而且大量的数据导致存储空间不足，这时侯。一台服务器已经不能满足需求，需要将应用和数据分离，整个网站使用三台服务器：应用服务器、文件服务器和数据库服务器。

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/Framework_2.jpg)

#### 