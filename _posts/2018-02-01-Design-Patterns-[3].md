---
layout: post
title:  "设计模式[3] -- 适配器模式"
categories: 设计模式
tags:  设计模式
author: G.Fukang
---
* content
{:toc}
适配器模式（结构型）实现：继承实现、组合实现

## 核心作用

将一个类的接口转换成客户希望的另外一个接口，适配器模式使得原本由于接口不兼容而不能在一起工作的那些类可以在一起工作。

**模式中的角色：**

- 目标接口 `Target`：客户所期待的接口，目标可以是具体的或抽象的类，也可以是接口
- 需要适配的类`Adaptee`：需要适配的类或者适配者类
- 适配器`Adapter`：通过包装一个需要适配的对象，把原接口抓换成目标接口


## 应用场景

- `java.io.InputStreamReader(InputStream)`
- `java.io.OutputStreamReader(OutputStream)`

## 适配器模式实现

需要被适配的类

```java
public class Adaptee {
    public void request(){
        System.out.println("可以完成客户请求的需要的功能");
    }
}
```

目标接口

```java
public interface Target {
    void handleReq();
}
```

适配器（继承）

```java
public class Adapter extends Adaptee implements Target{
    @Override
    public void handleReq() {
        super.request();
    }
}
```

适配器（组合）

```java
public class Adapter2 implements Target{
    private Adaptee adaptee;
    @Override
    public void handleReq() {
        adaptee.request();
    }

    public Adapter2(Adaptee adaptee){
        super();
        this.adaptee=adaptee;

    }
}
```

客户端

```java
public class Client {
    public void test1(Target target){
        target.handleReq();
    }

    public static void main(String[] args) {
        Client client=new Client();

        Target target=new Adapter();

        client.test1(target);

        Adaptee adaptee=new Adaptee();

        Target target1=new Adapter2(adaptee);

        client.test1(target1);
    }
}
```

关系图

![这里写图片描述](http://img.blog.csdn.net/20180201193851974?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYW5vbnltb3VzRw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)