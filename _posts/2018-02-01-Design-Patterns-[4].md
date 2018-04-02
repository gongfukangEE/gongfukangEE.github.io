---
layout: post
title:  "设计模式[4] -- 代理模式"
categories: 设计模式
tags:  设计模式 代理模式
author: G.Fukang
---
* content
{:toc}
代理模式（结构型）实现：静态代理和动态代理 

## 核心作用

- 通过代理，控制对象访问

  可以详细控制访问某个（某类）对象的方法，在调用这个方法前做前置处理，调用这个方法后做后置处理

- `AOP(Aspect Oriented Programming)`面向切面编程的核心实现机制

**个人理解**：

经纪公司会签约很多歌手，这就是**代理人**，当有人要歌手A（某个对象）唱歌的时候，会和经纪公司面谈、签合同、收预付款这就相当于**前置处理**，然后安排歌手唱歌（调用这个方法），歌手A唱完歌后，经纪公司再收尾款，这就相当于**后置处理**，在这个过程中，歌手A只负责唱歌，其他事情不需要操心，并且当安排经纪公司的其他歌手唱歌的时候也是这个流程。

**模式中的角色**：

- 抽象角色：定义代理角色和真实角色的公共对外方法
- 真实角色：实现抽象角色，定义真实角色所要实现的业务逻辑，供代理角色调用
- 代理角色：实现抽象角色，是真实角色的代理，通过真实角色的业务逻辑方法来实现抽象方法，并可以附加自己的操作

## 应用场景

- 安全代理：屏蔽对真实角色的访问
- 远程代理： 通过代理类处理远程方法调用
- 延迟加载： 先加载轻量级的代理对象，真正需要再加载真实对象

## 分类

- 静态代理
- 动态代理
  - `JDK`自带的动态代理
  - `javaassist`字节码操作库实现

## 代理模式实现

### 静态代理模式

抽象角色

```java
public interface Star {

    void sing();

    void collectMoney();
}
```

真实角色

```java
public class RealStar implements Star {

    @Override
    public void sing() {
        System.out.println("RealStar.sing()");
    }

    @Override
    public void collectMoney() {
        System.out.println("RealStar.collectMoney");
    }
}
```

代理角色

```java
public class ProxyStar implements Star {
    private Star reaslstar;

    public ProxyStar(Star s) {
        super();
        this.reaslstar = s;
    }

    @Override
    public void sing() {
       reaslstar.sing();
    }

    @Override
    public void collectMoney() {
        System.out.println("ProxyStar.collectMoney()");
    }
}
```

客户端

```java
public class Client {
    public static void main(String[] args) {
        Star realStar=new RealStar();
        Star proxySter=new ProxyStar(realStar);

        proxySter.sing();
        proxySter.collectMoney();
    }
}
```

### 动态代理模式

**优点**：抽象角色中（接口）声明的所有方法都被转移到调用处理器一个集中的方法中处理，这样我们就可以更加灵活和统一的处理众多的方法。

**`JDK`自带的动态代理：**

- `java.lang.reflect.Proxy`
  - 作用：动态生成代理类和对象
- `java.lang.reflect.InvocationHandler`
  - 可以通过`invoke`方法实现对真实角色的代理访问
  - 每次通过`Proxy`生成代理类对象时都啊哟指定对应的处理器对象

**关于`invoke`**

- Object proxy  **代理对象**
- Method method  **调用代理对象的某个方法的Method对象**
- Object[] args  **调用真实对象某个方法时接受的参数**

利用方法控制替换掉代理角色，实现动态代理

```java
public class StarHandler implements InvocationHandler {

    Star realStar;

    public StarHandler(Star realStar){
        super();
        this.realStar=realStar;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
       Object object=null;

        System.out.println("真正的方法执行前！");
        System.out.println("面谈，签合同，预付款，订机票");

        if(method.getName().equals("sing")){
            object=method.invoke(realStar,args);
        }

        System.out.println("真正的方法执行后！");
        System.out.println("收尾款");
        return object;
    }
}
```

客户端

```java
public class Client {
    public static void main(String[] args) {

        Star realStar = new RealStar();
        StarHandler handler = new StarHandler(realStar);

        Star proxy = (Star) Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(),
                new Class[]{Star.class}, handler);


        proxy.bookTicket();
        proxy.sing();

    }
}
```

