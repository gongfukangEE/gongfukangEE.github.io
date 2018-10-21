---
layout: post
title:  "设计模式[2] -- 工厂模式"
categories: 设计模式
tags:  工厂模式
author: G.Fukang
---
* content
{:toc}
工厂模式（创建型模式）实现：简单工厂模式、工厂方法模式、抽象工厂模式

## 核心作用

**实现创建者和调用者分离，实例化对象，用工厂方法替代`new`操作。将选择实现类、创建对象统一管理和控制。从而将调用者跟我们的实现类解耦**

## 应用场景

- `JDK`中`Calendar`的`getImstance()`方法
- `JDBC`中`Connection`对象的获取
- `Spring`中`IOC`容器创建管理`bean`对象
- 反射中`Class`对象的`newInstance()`
- `XML`解析时的`DocumentBuliderFactory`创建解析对象

## 非工厂模式示例

接口
```java
public interface bike {
    void run();
}
```
产品小黄车
```java
public class ofo implements bike {
    @Override
   public void run(){
        System.out.println("小黄车");
    }
}
```
产品摩拜单车
```java
public class mobike implements bike{
    public void run(){
        System.out.println("摩拜单车");
    }
}
```

生产

```java
public class client01 {
    public static void main(String[] args) {
        bike bike1=new ofo();
        bike bike2=new mobike();

        bike1.run();
        bike2.run();
    }
}
```

关系图

![这里写图片描述](http://img.blog.csdn.net/20180201110648752?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYW5vbnltb3VzRw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 工厂模式实现

### 简单工厂模式

工厂一般使用静态方法，通过接收参数的不同来返回不同的对象实例，但是无法增加新产品（不修改代码）

增加工厂类

```java
public class bikeFactory {
    public static bike createOfo(){
        return new ofo();
    }
    public static bike createMobike(){
        return new mobike();
    }
}
```

生产

```java
public class client02 {
    public static void main(String[] args) {
        bike bike01=bikeFactory.createOfo();
        bike bike02=bikeFactory.createMobike();

        bike01.run();
        bike02.run();
    }
}
```

关系图

![这里写图片描述](http://img.blog.csdn.net/20180201110700772?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYW5vbnltb3VzRw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 工厂方法模式

相对于简单工厂模式，工厂方法模式有一组实现了相同接口的工厂类

增加生产接口

```java
public interface bikeFactory {
    bike createBike();
}
```

实现产品小黄车生产

```java
public class ofoFactory implements bikeFactory {
    @Override
    public bike createBike(){
        return new ofo();
    }
}
```

实现产品摩拜单车生产

```java
public class mobikeFactory implements bikeFactory {
    @Override
    public bike createBike(){
        return new mobike();
    }
}
```

生成

```java
public class client {
    public static void main(String[] args) {
        bike bike01 = new ofoFactory().createBike();
        bike bike02 = new mobikeFactory().createBike();

        bike01.run();
        bike02.run();
    }
}
```

关系图

![这里写图片描述](http://img.blog.csdn.net/20180201110737495?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYW5vbnltb3VzRw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 抽象工厂模式

用来生产不同产品族的全部产品，不可增加新产品，但是可以增加新产品族

从部件到整体：

部件：

- Engine < interface >
  - highEngine < implements >

  - lowEngine < implements >

    代码片段

    ```java
    public interface Engine {
        void run();
        void start();
    }

    class highEngine implements Engine{
        @Override
        public void run(){
            System.out.println("转得快");
        }

        @Override
        public void start(){
            System.out.println("启动快");
        }
    }

    class lowEngine implements Engine{
        @Override
        public void run(){
            System.out.println("转得慢");
        }

        @Override
        public void start(){
            System.out.println("启动慢");
        }
    }
    ```

    ​
- Seat < interface >
  - highSeat < implements >

  - lowSeat < implements >

    代码片段

    ```java
    public interface Seat {
        void message();
    }

    class highSeat implements Seat{
        @Override
        public void message(){
            System.out.println("高级座椅");
        }
    }

    class lowSeat implements Seat{
        @Override
        public void message(){
            System.out.println("低端座椅");
        }
    }
    ```

- Type < interface >
  - highType < implements >

  - lowType < implements >

    代码片段

    ```java
    public interface Type {
        void revolve();
    }

    class highType implements Type{
        @Override
        public void revolve(){
            System.out.println("好轮胎");
        }
    }

    class lowType implements Type{
        @Override
        public void revolve(){
            System.out.println("差轮胎");
        }
    }
    ```

产品族

- highCarFactory < implements >

  - highEngine 

  - highSeat 

  - highType 

    代码片段

    ```java
    public class highCarFactory implements CarFactory {
        @Override
        public Engine createEngine(){
            return new highEngine();
        }

        @Override
        public Seat createSeat(){
            return new highSeat();
        }

        @Override
        public Type createType(){
            return  new highType();
        }
    }
    ```

- lowCarFactory < implements >
  - lowEngine 

  - lowSeat 

  - lowType

    代码片段

    ```java
    public class lowCarFacroty implements CarFactory{
        @Override
        public Engine createEngine(){
            return new lowEngine();
        }

        @Override
        public Seat createSeat(){
            return new lowSeat();
        }

        @Override
        public Type createType(){
            return  new lowType();
        }
    }
    ```

生产工厂

  CarFactory < interface >

  代码片段

  ```java
  public interface CarFactory {
      Engine createEngine();
      Seat createSeat();
      Type createType();
  }
  ```

生产线

  ```java
  public class client {
      public static void main(String[] args) {
          System.out.println("highCar");
          CarFactory carFactor=new highCarFactory();
          Engine engine=carFactor.createEngine();
          Seat seat=carFactor.createSeat();
          Type type=carFactor.createType();

          engine.run();
          engine.start();
          seat.message();
          type.revolve();

          System.out.println("lowCar");
          CarFactory carFactory01=new lowCarFacroty();
          Engine engine1=carFactory01.createEngine();
          engine1.run();
          engine1.start();
      }
  }
  ```

  ​

  ​

  ​

