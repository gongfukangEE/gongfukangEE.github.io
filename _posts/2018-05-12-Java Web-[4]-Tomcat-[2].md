---
layout: post
title:  "Java Web 学习笔记[4] -- Tomcat 的设计模式"
categories: [WEB开发, Java-Web, 设计模式, Tomcat]
description: Tomcat 的系统架构与设计模式（下）
keywords:  Tomcat, Java-Web
author: G.Fukang
---
## 概述

Tomcat 中用了很多设计模式，比如：模板模式，工厂模式和单例模式等常见的设计模式，除此之外还有以下几种设计模式。

## 门面设计模式

### 应用

- 在 Request 和 Response 对象封装
- 从 StandardWrapoper 到 ServlertConfig 封装
- 从 ApplicationContext 到 ServletContext 封装

### 原理

在一个大的系统中有多个子系统时，这时多个子系统肯定要相互通信，但是每个子系统又不能将自己的内部数据过多地暴露给其他系统，不然就没有必要划分子系统了，这个子系统就会设计一个门面，把别的系统感兴趣的数据封装起来，通过这个门面来进行访问。

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/%E9%97%A8%E9%9D%A2%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.jpg)

门面设计模式的关键就是：Client 只能访问 Facade 中提供的数据。

## 观察者设计模式

观察者设计模式通常也叫做发布 - 订阅模式，也就是事件的监听机制。

### 应用

- 控制组件生命周期的 Lifecycle
- 对 Servlet 实例的创建
- Session 的管理
- Container

### 原理

通俗来讲就是你做事情的时候有人盯着你，当你做的事情是他感兴趣的事情时，他就会跟着做另外一些事情，但是盯着你的人必须到你那里登记，不然你无法通知他。

**重要角色**

- Subject 抽象主题：它负责管理所有观察者的引用，同时定义主要的事件操作
- ConcreteSubject 具体主题：它实现了抽象主题定义的所有接口，当自己发生变化时，会通知所有观察者
- Observer 观察者：监听主题发生变化的操作接口

## 命令设计模式

### 原理

命令设计模式就是封装命令，把发出命令的责任和执行命令的责任分开，也就是一种功能的分工。不同的模块可以对同一个命令做出不同的解释

**重要角色**

- Client：创建一个命令，并决定接受者
- Command：命令接口，定义一个抽象方法
- ConcreteCommand：具体命令，负责调用接受者的相应操作
- Invoker：请求者，负责调用命令对象执行请求
- Receiver：接受者，负责具体实施和执行一次请求

### 应用

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/%E5%91%BD%E4%BB%A4%E6%A8%A1%E5%BC%8F.png)

Tomcat 中命令模式在 Connector 和 Container 组件之间有体现，Tomcat 作为应用服务器，在接收到很多请求的时候，如何分配和执行这些请求是必须的功能。

Connector 作为抽象请求者，HttpConnector 作为具体请求者，HttpProcessor 作为命令，Container 作为命令的抽象接受者，ContainerBase 作为具体的接受者，客户端就是应用服务器 Server 组件了。

Server 首先创建命令请求者 HttpConnector 对象，然后创建命令 HttpProcessor 命令对象，再把命令对象交给命令接受者 ContainerBase 容器来处理命令。命令的最终是被 Tomcat 的 Container 执行的。

## 责任链设计模式

### 原理

责任链模式就是很多对象由每个对象对其下家的引用而连接起来行成一条链，请求在这条链上传递，直到链上的某个对象处理此请求，或者每个对象都可以处理请求，并传给“下家”，直到最终链上每个对象都处理完。这样可以不影响客户端而能够在链上增加任意的处理节点。

**重要角色**

- Handler（抽象处理者）：定义一个处理请求的接口
- ConcreteHandler（具体处理者）：处理请求的具体类，或者传给“下家”

### 应用

Tomcat 容器设置就是责任链模式，从 Engine 到 Host 再到 Context 一直到 Wrapper 都通过一个链传递请求，对应的责任链模式的角色中 Container 扮演抽象处理者角色，具体处理者由 StandardEngine 等子容器扮演。



