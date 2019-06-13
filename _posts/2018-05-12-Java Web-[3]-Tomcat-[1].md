---
layout: post
title:  "Java Web 学习笔记[3] -- Tomcat 的系统架构"
categories: [WEB开发, Java-Web]
description: Tomcat 的系统架构与设计模式（上）
keywords:  Tomcat, Java-Web
author: G.Fukang
---


《深入分析 Java Web 技术内幕》读书笔记：Tomcat 的系统架构与设计模式（上）

## Tomcat 总体结构

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/tomcat%20%E6%80%BB%E4%BD%93%E7%BB%93%E6%9E%84.jpg)

Tomcat 的核心组件有两个：Connector 和 Container，其中一个 Container 对应多个 Connector。多个 Connector 和 一个 Container 就组成了一个 Service 对外提供服务，Service 的生存环境和生命周期由 Server 提供和控制。

### Serivce

在 Tomcat 中，将 Connector 和 Container 作为一个整体，Connector 负责对外交流，Container 负责处理 Connector 接收的请求，Service 只是在他们外面多包装了一层，将他们组装在一起，向外面提供服务。在 Tomcat 中 Service 接口的标准实现类是 StandardService，它不仅实现了 Service 接口，还实现了 Lifecycle 接口，可以控制组件的生命周期。其主要方法为：

`StandardService. SetContainer`

```java
public void setContainer(Container container) {
    Container oldContainer = this.container;
    if ((oldContainer != null) && (oldContainer instanceof Engine))
        ((Engine) oldContainer).setService(null);
    this.container = container;
    if ((this.container != null) && (this.container instanceof Engine))
        ((Engine) this.container).setService(this);
    if (started && (this.container != null) && (this.container instanceof Lifecycle)) {
        try {
            ((Lifecycle) this.container).start();
        } catch (LifecycleException e) {
            ;
        }
    }
    synchronized (connectors) {
        for (int i = 0; i < connectors.length; i++)
            connectors[i].setContainer(this.container);
    }
    if (started && (oldContainer != null) && (oldContainer instanceof Lifecycle)) {
        try {
            ((Lifecycle) oldContainer).stop();
        } catch (LifecycleException e) {
            ;
        }
    }
    support.firePropertyChange("container", oldContainer, this.container);
}
```

**setContainer**：先判断当前的这个 Service 有没有关联 Container，如果已经关联了，就去掉这个关联关系 oldContainer.setService(null) 。如果这个 oldContainer 已经启动了，则结束它的生命周期，然后再替换新的关联，再初始化并开始新的 Container 的生命周期，最后将这个过程通知感兴趣的事件监听程序

`StandardService. addConnector`

```java
public void addConnector(Connector connector) {
    synchronized (connectors) {
        connector.setContainer(this.container);
        connector.setService(this);
        Connector results[] = new Connector[connectors.length + 1];
        System.arraycopy(connectors, 0, results, 0, connectors.length);
        results[connectors.length] = connector;
        connectors = results;
        if (initialized) {
            try {
                connector.initialize();
            } catch (LifecycleException e) {
                e.printStackTrace(System.err);
            }
        }
        if (started && (connector instanceof Lifecycle)) {
            try {
                ((Lifecycle) connector).start();
            } catch (LifecycleException e) {
                ;
            }
        }
        support.firePropertyChange("connector", null, connector);
    }
}
```

**AddConnector**：首先设置关联关系，然后是初始化工作，开始新的生命周期

### Server

Server 要完成的任务很简单，就是要能够提供一个接口让其他程序能够访问这个 Service 集合、同时要维护它所包含的所有 Service 的生命周期，包括如何初始化，如何结束服务，如何找到别人访问的 Service

Server 的标准实现类是 StandardService，它也实现了 Lifecycle 其中一个重要方法是：addService

```java
public void addService(Service service) {
    service.setServer(this);
    synchronized (services) {
        Service results[] = new Service[services.length + 1];
        System.arraycopy(services, 0, results, 0, services.length);
        results[services.length] = service;
        services = results;
        if (initialized) {
            try {
                service.initialize();
            } catch (LifecycleException e) {
                e.printStackTrace(System.err);
            }
        }
        if (started && (service instanceof Lifecycle)) {
            try {
                ((Lifecycle) service).start();
            } catch (LifecycleException e) {
                ;
            }
        }
        support.firePropertyChange("service", null, service);
    }
}
```

从代码中可以看出，Service 和 Server 是相互关联的，Server 也是和 Service 管理 Connector 一样管理它，也是将 Service 放在一个数组中，后面部分的代码也是管理这个新加进来的 Service 的生命周期。

### Lifecycle

Tomcat 中组件的生命周期是通过 Lifecycle 接口来控制的，组件只要继承这个接口并实现其中的方法就可以统一被拥有它的组件控制，这样一层一层的直到一个最高级的组件就可以控制 Tomcat 中所有组件的生命周期，这个最高级的组件就是 Server，控制 Server 的是 Startup，也就是启动和关闭 Tomcat。

通俗来说 Tomcat 就是以容器的方式来组织整个系统架构的，反映到数据结构就是树，树的根节点没有父节点，其他节点有且仅有一个父节点，每个父节点有零个或者多个子节点。可以通过父容器启动它的子容器，这样只要启动根容器，就可以把所有容器都启动，达到统一启动、停止、关闭的效果。比如在 Server 中 Start 方法就会循环启动调用 Service 组件的 Start 方法。其中 `StandardServer.Start`源码如下

```java
public void start() throws LifecycleException {
    if (started) {
        log.debug(sm.getString("standardServer.start.started"));
        return;
    }
    lifecycle.fireLifecycleEvent(BEFORE_START_EVENT, null);
    lifecycle.fireLifecycleEvent(START_EVENT, null);
    started = true;
    synchronized (services) {
        for (int i = 0; i < services.length; i++) {
            if (services[i] instanceof Lifecycle)
                ((Lifecycle) services[i]).start();
        }
    }
    lifecycle.fireLifecycleEvent(AFTER_START_EVENT, null);
}
```

## Connector 组件

Connector 最重要的功能就是接收连接请求，然后分配线程让 Container 来处理这个请求，这必然是**多线程**的。

具体来说 Connector 负责接收浏览器发过来的 TCP 连接请求，创建一个 Request 和 Response 对象分别用于和请求端交换数据，然后会产生一个线程来处理这个请求并把产生的 Request 和 Response 对象传给处理这个请求的线程，而处理这个请求的线程就是 Container 组件要做的事情了。

## Servlet 容器 Container 

Container 是容器的父接口，所有子容器都必须实现这个接口，Container 容器的设计是**责任链**的设计模式，它有四个子容器组件构成：Engine、Host、Context、Wrapper。这四个子容器是父子关系，**前者包含后者**。

### Wrapper

Wrapper 代表一个 Servlet，它负责管理一个 Servlet，包括 Servlet 的装载，初始化，执行及资源回收

### Context

Context 代表 Servlet 的 Context，它具备了 Servlet 运行的基本环境，理论上只要有 Context 就能运行 Servlet。

Context 最重要的功能就是管理它里面的 Servlet 实例，Servlet 实例在 Context 中是以 Wrapper 出现的。关于Context 如何找到正确的 Servlet 并执行它，可以查看 [Servlet 调用](https://gongfukangee.github.io/2018/05/12/Java-Web-1-Servlet/#servlet-%E8%B0%83%E7%94%A8) 。

**Context 和 Wrapper 的处理请求时序图**

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/Tomcat%20Wrapper%26Context.png)

### Host 容器 & Engine 容器

Engine 容器比较简单，它只定义了一些基本的关联关系。

Host 是 Engine 的子容器，一个 Host 在 Engine 中代表一个虚拟主机，这个虚拟主机的作用就是运行多个应用，它负责安装和展开这些应用，并且标识这个应用以便于能够区分它们。它的子容器是 COntext，它除了关联子容器外还保存一个主机应有的信息。

**Engine 和 Host 处理请求的时序图**

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/Tomcat%20Host%26Engine.png)




