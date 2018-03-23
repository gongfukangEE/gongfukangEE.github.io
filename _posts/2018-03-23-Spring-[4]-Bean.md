---
layout: post
title:  "Spring学习笔记[4] -- Bean"
categories: WEB开发
tags:  Spring
author: G.Fukang
---
* content
{:toc}
Spring学习之Bean



## 继承

Spring允许定义一个父`<bean>`，子`<bean>`将自动继承父`<bean>`的配置信息

```xml
<bean id="car1" class="gongfukangee.Demo.Car" 
      P:brand="红旗" p:price="2000" p:color="黑色"/>
<bean id="car2" class="gongfukangee.Demo.Car" 
      P:brand="红旗" p:price="2000" p:color="红色"/>

<!-- 继承方式实现 -->
<!-- 定义为抽象bean -->
<bean id="abstratCar" class="gongfukangee.Demo.Car"
      p:brand="红旗" p:price="2000" p:color="黑色" abstract="true"/>
<!-- 继承 abstractCar -->
<bean id="car3" p:color="蓝色" parent="abstractCar"/>
<bean id="car3" p:brand="奔驰" parent="abstractCar"/>
```

## 依赖

Spring允许用户通过`depends-on`属性显示指定`Bean`前置依赖的`Bean`，前置依赖的`Bean`会在本`Bean`实例化之前就创建好，例如：保证`manager`在实例化之前，`sysInit`就已经初始化完成

```xml
<bean id="manager" class="gongfukang.Demo.CacheManager" depends-on="sysInit"/>
<bena id="sysInit" class="gongfukang.Demo.SysInit"/>
```

## 引用

假如一个`<bean>`要引用另一个`<bean>`的`id`属性值，则可以直接配置：

```xml
<bean id="car" class="gongfukangee.Demo.Car"/>
<bean id="boss" class="gongfukangee.Demo.Boss"
      p:carId="car" scope="prototype"/>
```

一般情况下，在一个`Bean`中引用另一个`Bean`的`id`是希望在运行期通过`getBean(beanName)`方法获取对应的`Bean`，但是由于Spring并不会在容器启动的时候对属性配置进行特殊的检查，因此即使拼写错误，也需要等到具体调用的时候才会发现。可以同过Spring的`<idref>`元素标签来优化配置

```xml
<bean id="car" class="gongfukangee.Demo.Car"/>
<bean id="boss" class="gongfukangee.Demo.Boss">
    <property name="carId">
        <idref bean="car"/>
    </property>
</bean>
```

