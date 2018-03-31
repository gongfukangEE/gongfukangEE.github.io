---
layout: post
title:  "Spring学习笔记[8] -- Cache[缓存]"
categories: WEB开发
tags:  Spring
author: G.Fukang
---
* content
{:toc}
Spring学习之缓存

## 缓存概述

缓存可以定义为一种存储机制，它将数据保存在某个地方，并以一种更快的方式提供服务。

比如：在从数据库查询用户的时候，首次被查询的用户将信息保存在缓存里，在用户再次被查询的时候可以直接从缓存获取信息，更加快速。

## 缓存基本概念

### 缓存命中率

即从缓存中读取的次数与总读取次数的比率

命中率=从缓存中读取的次数/(总读取次数[从缓存读取的次数+从慢速设备读取的次数])

### 过期策略

- FIFO：先进先出策略，即先放入缓存的数据最先被移除
- LRU：未久使用策略，即使用时间距离现在最久的那个数据被移除
- LFU：最近最少使用策略，即一定时间段内使用次数最少的那个数据被移除

## Spring Cache

### 优点

- 支持开箱即用，并提供基本的Cache抽象，方便切换各种底层Cache
- 类似于Spring提供的事务管理，通过Cache注解即可实现缓存逻辑透明化
- 当事务回滚，缓存也会自动回滚
- 支持比较复杂的缓存逻辑

### 缺点

- Spring Cache 并不针对多线程的应用环境进行专门处理
- Spring Cache抽象的操作没有锁的概念，当多线程并发操作同一个缓存项时候，将可能得到过期的数据

## 自定义缓存实现

**User实体类**

```java
public class User implements Serializable {
    private String userId;
    private String userName;
    private int age;
    ...get&set...
}
```

**缓存管理器**

```java
public class CacheManager<T> {
    private Map<String,T> cache=new ConcurrentHashMap<String,T>();
    public T getValue(Object key){
        return cache.get(key);
    }
    public void addOrUpdateCache(String key,T value){
        cache.put(key,value);
    }
    //根据key来删除缓存中的一条记录
    public void evictCache(String key){
        if(cache.containsKey(key))
            cache.remove(key);
    }
    //清空缓存中的记录
    public void evictCache(){
        cache.clear();
    }
}
```

**用户查询服务类**

```java
public class UserService {
    private CacheManager<User> cacheManager;
    public UserService(){
        //构造一个缓存管理器
        cacheManager=new CacheManager<User>();
    }
    public User getUserById(String userId){
        //首先查询缓存
        User result=cacheManager.getValue(userId);
        if(result!=null){
            System.out.println("get from cache...."+userId);
            //如果在缓存中，直接返回缓存的结果
            return result;
        }
        //否则到数据库中查询
        result=getFromDB(userId);
        if(result!=null){
            //将查询结果更新到缓存中
            cacheManager.addOrUpdateCache(userId,result);
        }
        return result;
    }
    public void reload(){
        cacheManager.evictCache();
    }
    private User getFromDB(String userId){
        System.out.println("real querying db..."+userId);
        return new User(userId);
    }
}
```

**测试**

```java
public class UserMain {
    public static void main(String[] args) {
        UserService userService=new UserService();
        //开始查询账号
        //第一次从数据库查询
        userService.getUserById("001001");
        //第二次直接从缓存返回
        userService.getUserById("001001");
        //缓存重置
        userService.reload();
        System.out.println("after reload...");
        //再次查询
        userService.getUserById("001001");
        userService.getUserById("001001");
    }
}
```

## Spring Cache 改进

Spring已经提供了默认的缓存管理器，所以只需要改造 `UserService`和`UserMain`

**用户查询服务类**

```java
@Service(value = "userServiceBean")
public class UserService {
    //使用一个名为 users 的缓存
    @Cacheable(cacheNames = "users")
    public User getUserById(String userId){
        //方法内部实现不考虑缓存逻辑，直接实现业务
        System.out.println("real query user."+userId);
        return getFromDB(userId);
    }
    private User getFromDB(String userId){
        System.out.println("real querying db..."+userId);
        return new User(userId);
    }
}
```

**XML配置文件**

```xml
    <cache:annotation-driven/>
    <bean id="userServiceBean" class="main.Demo02.Service.UserService"/>
    <bean id="cacheManager" class="org.springframework.cache.support.SimpleCacheManager">
        <property name="caches">
            <set>
                <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="default"/>
                <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="users"/>
            </set>
        </property>
    </bean>
```

**测试**

```java
public class UserMain {
    public static void main(String[] args) {
        ApplicationContext context=new ClassPathXmlApplicationContext("applicationContext.xml");
        UserService userService=(UserService) context.getBean("userServiceBean");
        //第一次查询
        System.out.println("first query...");
        userService.getUserById("somebody");
        //第二次查询
        System.out.println("second query...");
        userService.getUserById("somebody");
    }
}
```

## 缓存注解

### @CaCheable

它指定了被注解方法的返回值是可被缓存的，其工作原理是Spring首先在缓存中查找数据，如果没有则执行方法并缓存结果，然后返回数据。

```java
@Cacheable(cacheNames = "users")
```

还可以以列表的形式提供多个缓存

```java
@Cacheable(cacheNames={"cache1","cache2"})
```

- 键生成器

  缓存的本质就是键/值对集合。Spring 4.0 默认提供了`SimpleKeyGenerator`生成器，其生成规则：

  - 如果没有方法入参，则使用`SimpleKey.EMPTY`作为key
  - 如果只有一个入参，则使用该入参作为Key
  - 如果多个入参，则返回包含所有入参的一个`SimpleKay`

  源码：

  ```java
  public static Object generatorKey(Object... params){
      if(params.length==0)
          return SimpleKay.EMPTY;
      if(params.length==1){
          Object param=params[0];
          if(param!=null && param.getClass().isArray())
              return param;
      }
      return new SimpleKey(params);
  }

  ```

- 带条件的缓存

  `condition`属性使用了SpEL表达式动态评估方法入参是否满足缓存条件

  ```java
  @Cacheable(cacheNames="users",condition="#user.age<35")
  ```

### 其他

- @CachePut 同 @Cacheable
- @CacheEvict 是@Cacheable注解的反向操作
- @Caching 组注解，可以为一个方法定义提供基于@Cacheable、@CacheEvict或者@CachePu注解的数组

## 基于XML的Cache声明

```xml
<!-- 定义需要使用缓存的类 -->
<bean id="userService" class="main.Demo02.Service.UserService"/>
<!-- 缓存定义 -->
<cache:advice id="cacheAdvice" cache-manager="cacheManager">
    <cache:caching cache"users">
        <cache:cacheable method="findUser" key="#userId"/>
        <cache:cache-evict method="loadUsers" all-entries="true"/>
    </cache:caching>
</cache:advice>
<aop:config>
    <aop:advisor addvice-ref="cacheAdvice" 
                 pointcut="execution(*main.Demo02.Service.UserService.*(..))"/>
</aop:config>
```








