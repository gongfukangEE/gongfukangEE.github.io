---
layout: post
title:  "Java Web 学习笔记[2] -- 深入理解 Session 与 Cookie"
categories: [WEB开发, Java-Web]
description: 深入理解 Session 与 Cookie
keywords:  Session, Cookie, Java-Web
author: G.Fukang
---
## 理解 Cookie

### Cookie 作用

当一个用户通过 HTTP 访问一个服务器时，这个服务器会将一些 Key/Value 键值对返回给客户端浏览器，并且给这些数据加上一些限制条件，在条件符合的时候这个用户下次访问这个服务器时，数据又被完整得带回给服务器。

### Cookie 属性

Cookie 有两个版本：Version 0 和 Version 1，它们有两种设置响应头的标识，分别是 Set-Cookie 和 Set-Cookie2

**Version 0**

| 属性项     | 介绍                                                         |
| ---------- | ------------------------------------------------------------ |
| NAME=VALUE | 键值对，可以设置保存 Key/value                               |
| Expires    | 过期时间，在设置的某个时间点后该 Cookie 失效                 |
| Domain     | 生成该 Cookie 的域名                                         |
| Path       | 该 Cookie 是在当前哪个路径下生成的                           |
| Secure     | 如果设置了该属性，那么只会在 SSH 连接的时候才会回传该 Cookie |

**Version 1**，仅与 Version 0 不同属性项

| 属性项     | 介绍                                 |
| ---------- | ------------------------------------ |
| Comment    | 注释项                               |
| CommentURL | 服务器为此 Cookie 提供的 URI 注释    |
| DIscard    | 是否在会话结束后丢弃该 Cookie 项     |
| Max-Age    | 最大失效时间，设置的是在多少秒后失效 |
| Port       | 该 Cookie 在什么端口下可以回传服务端 |

### Cookie 如何工作

**服务端创建 Cookie**

Tomcat 创建 Set-Cookie 响应头的时序图

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/Web/tomcat-cookie.jpg)

真正构建 Cookie 是在 Response 类中完成的，调用 generateCookieString 方法将 Cookie 对象构造成一个字符串，构造的字符串格式如 `userName = "gfk"; Version = "1"; Domain = "gongfukang.net"; Max-Age = 1000`，然后将这个字符串命名为 Set-Cookie 添加到 MimeHeaders 中。 

**客户端获取 Cookie**

当我们请求某个 URL 路径时，浏览器会根据这个 URL 路径将符合条件的 Cookie 放在 Request 请求头中传回给服务端，服务端通过 request.getCookie() 来取得所有 Cookie

### 使用 Cookie 的限制

Cookie 是 HTTP 头中的一个字段，虽然 HTTP 本身对这个字段并没有多少限制，但是 Cookie 最终还是存储在浏览器里，对 Cookie 的大小和数量还是有所限制的。

## 理解 Session

### 为什么使用 Session

同一个客户端每次和服务端交互的时候，不需要每次都传回所有的 Cookie 值，而是只需要传回一个 ID，这个 ID 是客户端第一次访问服务器所生成的，而且每个客户端是唯一的，这样每个客户端就有了一个唯一的 ID，客户端只要传回这个 ID 就行了。

### Session 与 Cookie 关系

Cookie 和 Session 的方案虽然分别属于客户端和服务端，但是服务端的 Session 的实现对客户端的 Cookie 有依赖关系。服务端执行 Session 机制时候会生成 Session 的 ID 值，这个 ID 值会发送给客户端，客户端每次请求都会把这个 ID 值放到 HTTP 请求的头部发送给服务端，而这个 ID 值在客户端会保存下来，保存的容器就是 Cookie。

Session 基于 Cookie 三种工作方式：

- 基于 URL Path Parameter，默认支持
- 基于 Cookie，如果没有修改 Context 容器的 Cooikes 标识，则默认也是支持的
- 基于 SSL，默认不支持

### Session 工作机制

有了 Session ID，服务端就可以创建 Sessin 对象了，第一次触发通过调用 request.getSession() 方法，如果当前的 Session ID 还没有对应的 HttpSession 对象，那么就创建一个新的，并将这个对象加到 org.apache.catalina.Manager 的 session 容器中保存。Manager 类将管理所有的 Session 的生命周期，Session 过期将被回收，服务器关闭，Session 将被序列化到磁盘等。只要这个 HttpSession 对象存在，用户就可以根据 Session ID 来获取这个对象，也就做到了对状态的保持。

## Cookie 和 Session 区别

**Cookie 和 Session 都是用来跟踪浏览器用户身份的会话方式**

### 概述

- Session 是在**服务端**保存的一个数据结构，用来跟踪用户的状态，这个数据可以保存在集群，数据库，文件中
- Cookie 是**客户端**保存用户信息的一种机制，用来记录用户的一些信息，也是实现 Session 的一种方式

### 详解

**保存方式不同**

Cookie 数据是保存在客户端的，Session 数据是保存在服务端的

在登陆一个网站的时候，如果 web 服务器端使用的是 Session，那么所有的数据都保存在服务器上，客户端每次请求服务器的时候会发送当前会话的 Session ID，服务器根据当前Session ID 判断相应的用户数据标志，以确定用户是否登录或具有某种权限；如果浏览器使用的是 Cookie，那么所有的数据都保存在浏览器端，比如你登录以后，服务器设置了 Cookie 用户名，那么当你再次请求服务器的时候，浏览器会将用户名一块发送给服务器，能够保持长时间不掉线。

**隐私策咯不同**

Cookie 存储在客户端中，对客户端是可见的，客户端的一些程序可能会窥探，复制以至修正 Cookie 中的内容。

Session 由于存储在服务器上，对客户端是透明的，因此不存在敏感信息泄露的风险。

当选用 Cookie 存储敏感信息时，最好是将 Cookie 信息加密，提交到服务端再进行解密，提高安全性。

**有效期不同**

Cookie 的过期时间属性可以设置为一个很大的数字，可以保证持久地记载用户的信息。

Session 默认过期时间设置是 -1，自需要关闭了阅读器 Session 就会失效。如果设置 Session 过期时间过长，服务器累计的 Session 就会越多，容易招致内存溢出。

**服务器压力不同**

Session 是保管在服务器端的，每个用户都会产生一个 Session。假如并发访问的用户十分多，会产生十分多的 Session，耗费大量的内存。

Cookie 保管在客户端，不占用服务器资源。假如并发阅读的用户十分多，Cookie 是很好的选择。

**跨域支持不同**

Cookie 支持跨域名访问，Session 不支持

### 根据场景来理解

如购物车场景，当你点击下单按钮的时候，由于 HTTP 协议无状态，所以并不知道是哪个用户在操作，因此服务端需要为特定的用户创建特定的 Session，用于标识这个用户，并跟踪用户，这样才知道购物车里有什么东西。具体到服务端如何识别特定的用户的时候就需要 Cookie 了，每次 HTTP 请求的时候，客户端都会发送相应的 Cookie 信息到服务端。实际上大多数的应用都是用 Cookie 来实现 Session 跟踪的，第一次创建 Session 的时候，服务端会在 HTTP 协议中告诉客户端，需要在 Cookie 里面记录一个Session ID，以后每次请求把这个会话 ID 发送到服务器，我就知道你是谁了。

