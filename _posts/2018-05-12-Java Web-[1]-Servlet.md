---
layout: post
title:  "Java Web 学习笔记[1] -- Servlet 工作原理解析"
categories: [WEB开发, Java-Web]
description: Servlet 工作原理解析
keywords:  Servlet, Java-Web
author: G.Fukang
---
## Servlet  架构 

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/servlet-arch.jpg)

### Servlet  主要任务

- 读取客户端（浏览器）发送的显式的数据
- 读取客户端（浏览器）发送的隐式的 HTTP 请求数据
- 处理数据并生成结果
- 发送显式的数据（即文档）到客户端（浏览器）
- 发送隐式的 HTTP 响应到客户端（浏览器）

## Servlet 生命周期

Servlet 程序是由 web 服务器调用，web 服务器实现了对 servlet 生命周期的管理。

### 周期原理

```java
容器加载 -> 初始化 init (仅一次) -> 进入服务 service （Get/Post 请求）-> 销毁 destroy -> 容器卸载
```

- 创建 Servlet 实例

  web 容器负责加载 Servlet，当 web 容器启动时或者是在第一次使用这个 Servlet 时，容器会负责创建 Servlet 实例，但是用户必须通过部署描述符`web.xml`指定 Servlet 的位置，也就是 Servlet 所在的类名称，成功加载后，web 容器会通过**反射**的方式对 Servlet 进行实例化。

- web 容器调用 Servlet 的 init 方法，对 Servlet 进行初始化

  init 主要是为了让 servlet 对象在处理客户请求前可以完成一些初始化的工作，例如：建立数据库连接，获取配置信息等。同时，servlet 容器通过 init 方法的 ServletConfig 参数向 Servlet 传递配置信息。

- Servlet 初始化后，将一直存在容器中，service() 响应客户端请求

  每当一个客户请求一个 HttpServlet 对象，该对象的 Service() 方法就要调用，而且传递给这个方法一个“请求”（ServletRequest）对象和一个“响应”（ServletResponse）对象作为参数。默认的服务功能是调用与HTTP 请求的方法相应的 doGet 或者 doPost 功能。容器会构造一个表示客户端请求信息的请求对象（类型为ServletRequest）和一个用于对客户端进行响应的响应对象（类型为ServletResponse）作为参数传递给 service() 方法。在 service() 方法中，Servlet 对象通过 ServletRequest 对象得到客户端的相关信息和请求信息，在对请求进行处理后，调用 ServletResponse 对象的方法设置响应信息。

- Servlet 的销毁

  当容器检测到一个 Servlet 对象应该从服务中被移除的时候，容器会调用该对象的 destroy() 方法，以便让 Servlet 对象可以释放它所使用的资源，保存数据到持久存储设备中。在 destroy() 方法调用之后，容器会释放这个 Servlet 对象，在随后的时间内，该对象会被Java的垃圾收集器所回收。

## Servlet 运行工作原理

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/servlet-work.jpg)

### 工作流程

- Web Client 向 Servlet 容器（Tomcat）发出 Http 请求
- Servlet 容器接收 Web Client 的请求
- Servlet 容器创建一个 HttpRequest 对象，将 Web Client 请求的信息封装到这个对象中。
- Servlet 容器创建一个 HttpResponse 对象
- Servlet 容器调用 HttpServlet 对象的 service 方法，把 HttpRequest 对象与 HttpResponse 对象作为参数传给 HttpServlet 对象。
- HttpServlet 调用 HttpRequest 对象的有关方法，获取 Http 请求信息。
- HttpServlet 调用 HttpResponse 对象的有关方法，生成响应数据。
- Servlet 容器把 HttpServlet 的响应结果传给 Web Client。

### 请求处理流程

- 用户点击一个链接，指向了一个 servlet 而不是一个静态页面。
- 容器“看出”这个请求是一个 Servlet，所以它创建了两个对象 HttpServletRequest 和 HttpServletResponse。
- 容器根据请求中的URL找到正确的 Servlet，为这个请求创建或分配一个线程，并把请求和响应对象传递给这个Servlet 线程。
- 容器调用 Servlet 的 service() 方法。根据请求的不同类型，service() 方法会调用 doGet() 或 doPost() 方法。这里假设调用 doGet() 方法。
- doGet() 方法生成动态页面，并把这个页面“塞到”响应对象里，需要注意的是，容器还有响应对象的一个引用！
- 线程结束，容器把响应对象转换为一个HTTP响应，并把它发回给客户，然后删除请求和响应对象。

## Servlet 体系结构

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/servlet%20%E9%A1%B6%E5%B1%82%E7%B1%BB%E5%85%B3%E8%81%94%E5%9B%BE.png)

Servlet 就是基于这几个类运转的，与 Servlet 主动关联的三个类分别是：ServletConfig，ServletRequest，ServletResponse。这三个类都是通过容器传递给 Servlet 的，其中 ServletConfig 在 Servlet 初始化的时候就传给 Servlet 了，而后面两个是在请求到达时调用 Servlet 传递过来的。 

## Servlet 调用

用户发起一个请求的时候通常会包含如下信息：`http://hostname:port/contextpath/servletpath`，其中 hostname 和 port 用来与服务器建立 TCP 连接，后面的 URL 用来选择在服务器中哪个子容器服务用户的请求。

在 Tomcat 7 中，根据 URL 来找到 Servlet 容器的映射工作由类 tomcat.util.http.mapper 来完成。这个保存了 Tomcat 的 Container 容器中的所有子容器的信息，Request 类在进入 Container 容器之前，Mapper 将会根据这次请求的 hostname 和 contextpath 将 host 和context 容器设置到 Request 的 mappingDate 属性中。

Mapper 获取容器的完整关系：MapperListener 的 init 方法将 MapperListener 类作为一个监听者加到整个 Container 容器的每个子容器中，这样只要一个容器发生变化，MapperListener 都将会被通知到，相应的保存容器关系的 MapperListener 的 mapper 属性也会被修改。

## Servlet 线程安全 

当多个客户端并发访问同一个Servlet时，web服务器会为每一个客户端的访问请求创建一个线程，并在这个线程上调用service方法。

- Servlet 是**单实例多线程**的，如果存在可以修改的成员变量将会出现线程安全问题
- 使用 Servlet 最好保证 Servlet 是无状态的，也就是没有可以修改的成员变量

## Listener 监听器

Listener 的设计为开发 Servlet 应用程序提供了一种快捷的手段，能够方便地i从另一个纵向维度控制程序和数据，它是基于**观察者模式**设计。

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/servlet-Listener.png)

## Filter 过滤器

Filter 存在的意义就好比你要去成都，它是你的目的地，但是提供一个机制让你去的途中可以做一些拦截的工作。

Filter 提供了 request、response 和 FilterChain 对象（用于控制请求的流转）。

Filter 类的核心还是传递 FilterChain 对象，这个对象保存了到最终 Servlet 对象的所有 Filter 对象，这些对象都保存在 ApplicationFilterChain 对象的 filters 数组中。在 FilterChain 链上每执行一个 Filter 对象，数组的当前计数都会加 1 ，直到计数等于数组的长度，当 FilterChain 上所有的 Filter 对象执行完成后，就会执行最终的 Servlet。它是一种**责任链设计模式** 

## url-patten

在前面了解到一个请求最终会通过 Mapper 完成 Servlet 的分配，同样 Mapper 类会根据请求的 URL 来匹配在每个 Servlet 中配置的 url-patten。

Filter 的 url-patten 匹配是在创建 ApplicationFilterChain 对象时进行的，它会把所有定义的 Filter 的 url-patten 与当前的 URL 匹配，如果成功就将这个 Filter 保存到 filters 数组中，然后在 FilterChain 中依次调用。

url-appen 解析规则：

- 精确匹配：/foo.htm 只会匹配 foo.htm 这个 URL
- 路径匹配：/foo/* 会匹配以 foo 为前缀的 URL
- 后缀匹配：*.htm 会匹配所以以 .htm 为后缀的 URL

## Servlet 是如何工作的

当 Servlet 容器（比如 Tomcat）启动后，会部署和加载所有的 web 应用。当 web 应用被加载，Servlet 容器会创建一次 ServletContext，然后将其保存在服务器的内存中。web 应用的 web.xml 被解析，找到其中所有 servlet、filter 和 Listener 或者 @WebServlet、@WebFilter 和 @WebListener 注解的内容，创建一次并保存到服务器的内存中。对于所有过滤器会立即调用 init()。当 Servlet 容器停止，将卸载所有 web 应用，调用所有初始化的 Servlet 和 过滤器的 destroy() 方法，最后回收 ServletContext 和所有 Servlet、Filter 与 Listener 实例。