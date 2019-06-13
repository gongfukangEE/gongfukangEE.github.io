---
layout: post
title:  "如何设计一个秒杀系统"
categories: [秒杀, 并发, 架构]
tags:  秒杀 并发 架构
author: G.Fukang
---
如何设计一个秒杀系统

## 系统的特点

- 高性能：秒杀涉及大量的并发读和并发写，因此支持高并发访问这点非常关键
- 一致性：秒杀商品减库存的实现方式同样关键，有限数量的商品在同一时刻被很多倍的请求同时来减库存，在大并发更新的过程中都要保证数据的准确性。
- 高可用：秒杀时会在一瞬间涌入大量的流量，为了避免系统宕机，保证高可用，需要做好流量限制

## 优化思路

- 后端优化：将请求尽量拦截在系统上游
  - 限流：屏蔽掉无用的流量，允许少部分流量走后端。假设现在库存为 10，有 1000 个购买请求，最终只有 10 个可以成功，99% 的请求都是无效请求
  - 削峰：秒杀请求在时间上高度集中于某一个时间点，瞬时流量容易压垮系统，因此需要对流量进行削峰处理，缓冲瞬时流量，尽量让服务器对资源进行平缓处理
  - 异步：将同步请求转换为异步请求，来提高并发量，本质也是削峰处理
  - 利用缓存：创建订单时，每次都需要先查询判断库存，只有少部分成功的请求才会创建订单，因此可以将商品信息放在缓存中，减少数据库查询
  - 负载均衡：利用 Nginx 等使用多个服务器并发处理请求，减少单个服务器压力
- 前端优化：
  - 限流：前端答题或验证码，来分散用户的请求
  - 禁止重复提交：限定每个用户发起一次秒杀后，需等待才可以发起另一次请求，从而减少用户的重复请求
  - 本地标记：用户成功秒杀到商品后，将提交按钮置灰，禁止用户再次提交请求
  - 动静分离：将前端静态数据直接缓存到离用户最近的地方，比如用户浏览器、CDN 或者服务端的缓存中
- 防作弊优化：
  - 隐藏秒杀接口：如果秒杀地址直接暴露，在秒杀开始前可能会被恶意用户来刷接口，因此需要在没到秒杀开始时间不能获取秒杀接口，只有秒杀开始了，才返回秒杀地址 url 和验证 MD5，用户拿到这两个数据才可以进行秒杀
  - 同一个账号多次发出请求：在前端优化的禁止重复提交可以进行优化；也可以使用 Redis 标志位，每个用户的所有请求都尝试在 Redis 中插入一个 `userId_secondsKill` 标志位，成功插入的才可以执行后续的秒杀逻辑，其他被过滤掉，执行完秒杀逻辑后，删除标志位
  - 多个账号一次性发出多个请求：一般这种请求都来自同一个 IP 地址，可以检测 IP 的请求频率，如果过于频繁则弹出一个验证码
  - 多个账号不同 IP 发起不同请求：这种一般都是僵尸账号，检测账号的活跃度或者等级等信息，来进行限制。比如微博抽奖，用 iphone 的年轻女性用户中奖几率更大。通过用户画像限制僵尸号无法参与秒杀或秒杀不能成功

## 代码优化

代码整体思路参考的 [@crossoverJie](<https://github.com/crossoverJie>)，做了以下几点变动

1. 将 SSM 换成 SpringBoot，开箱即用，更加容易上手，将 Mapper 替换为注解，精简了部分代码
2. 原项目中依赖了开发者自己的开源包 [distributed-redis-tool](<https://github.com/crossoverJie/distributed-redis-tool>)，本项目将用到的限流部分直接集成到代码中
3. 摒弃了 Dubbo 和 Zookeeper，减少了代码量，也使得新手更加容易上手
4. 加入缓存预热，在秒杀开始前，将库存信息读到缓存中
5. 缓存更新逻辑中加入 Redis 事务

### Jmeter 压测并发量变化图

// TODO

### 0. 基本秒杀逻辑

```java
@Override
public int createWrongOrder(int sid) throws Exception {
    // 数据库校验库存
    Stock stock = checkStock(sid);
    // 扣库存(无锁)
    saleStock(stock);
    // 生成订单
    int res = createOrder(stock);
    return res;
}
private Stock checkStock(int sid) throws Exception {
    Stock stock = stockService.getStockById(sid);
    if (stock.getCount() < 1) {
        throw new RuntimeException("库存不足");
    }
    return stock;
}
private int saleStock(Stock stock) {
    stock.setSale(stock.getSale() + 1);
    stock.setCount(stock.getCount() - 1);
    return stockService.updateStockById(stock);
}
private int createOrder(Stock stock) throws Exception {
    StockOrder order = new StockOrder();
    order.setSid(stock.getId());
    order.setName(stock.getName());
    order.setCreateTime(new Date());
    int res = orderMapper.insertSelective(order);
    if (res == 0) {
        throw new RuntimeException("创建订单失败");
    }
    return res;
}
// 扣库存 Mapper 文件
@Update("UPDATE stock SET count = #{count, jdbcType = INTEGER}, name = #{name, jdbcType = 			     VARCHAR}, " + "sale = #{sale,jdbcType = INTEGER},version = #{version,jdbcType = INTEGER} " + "WHERE id = #{id, jdbcType = INTEGER}")
```

### 1. 乐观锁更新库存，解决超卖问题

超卖问题出现的场景

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/%E7%A7%92%E6%9D%80%E8%B6%85%E5%8D%96.png)

悲观锁是表级锁，虽然可以解决超卖问题，但是由于排斥外部请求，会导致很多请求等待锁，卡死在这里，如果这种请求很多就会耗尽连接，系统出现异常。乐观锁默认不加锁，跟新失败就直接返回抢购失败，可以承受较高并发

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/%E4%B9%90%E8%A7%82%E9%94%81%E6%89%A3%E5%BA%93%E5%AD%98.png)

```java
@Override
public int createOptimisticOrder(int sid) throws Exception {
    // 校验库存
    Stock stock = checkStock(sid);
    // 乐观锁更新
    saleStockOptimstic(stock);
    // 创建订单
    int id = createOrder(stock);
    return id;
}
// 乐观锁 Mapper 文件
@Update("UPDATE stock SET count = count - 1, sale = sale + 1, version = version + 1 WHERE " +
        "id = #{id, jdbcType = INTEGER} AND version = #{version, jdbcType = INTEGER}")
```

### 2. Redis 计数限流

根据前面的优化分析，假设现在有 10 个商品，有 1000 个并发秒杀请求，最终只有 10 个订单会成功创建，也就是说有 990 的请求是无效的，这些无效的请求也会给数据库带来压力，因此可以在在请求落到数据库之前就将无效的请求过滤掉，将并发控制在一个可控的范围，这样落到数据库的压力就小很多

关于限流的方法，可以看这篇博客[浅析限流算法](<https://gongfukangee.github.io/2019/04/04/Limit/>)，由于计数限流实现起来比较简单，因此采用计数限流，限流的实现可以直接使用 Guava 的 RateLimit 方法，但是由于后续需要将实例通过 Nginx 实现负载均衡，这里选用 Redis 实现分布式限流

在 `RedisPool` 中对 `Jedis` 线程池进行了简单的封装，封装了初始化和关闭方法，同时在 `RedisPoolUtil` 中对 Jedis 常用 API 进行简单封装，每个方法调用完毕则关闭 Jedis 连接。

限流要保证写入 Redis 操作的原子性，因此利用 Redis 的单线程机制，通过 LUA 脚本来完成。

![](https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/%E7%A7%92%E6%9D%80%E9%99%90%E6%B5%81.png)

```java
@Slf4j
public class RedisLimit {

    private static final int FAIL_CODE = 0;

    private static Integer limit = 5;

    /**
     * Redis 限流
     */
    public static Boolean limit() {
        Jedis jedis = null;
        Object result = null;
        try {
            // 获取 jedis 实例
            jedis = RedisPool.getJedis();
            // 解析 Lua 文件
            String script = ScriptUtil.getScript("limit.lua");
            // 请求限流
            String key = String.valueOf(System.currentTimeMillis() / 1000);
            // 计数限流
            result = jedis.eval(script, Collections.singletonList(key), Collections.singletonList(String.valueOf(limit)));
            if (FAIL_CODE != (Long) result) {
                log.info("成功获取令牌");
                return true;
            }
        } catch (Exception e) {
            log.error("Limit 获取 Jedis 实例失败：", e);
        } finally {
            RedisPool.jedisPoolClose(jedis);
        }
        return false;
    }
}
// 在 Controller 中，每个请求到来先取令牌，获取到令牌再执行后续操作，获取不到直接返回 ERROR
public String createOptimisticLimitOrder(HttpServletRequest request, int sid) {
    int res = 0;
    try {
        if (RedisLimit.limit()) {
            res = orderService.createOptimisticOrder(sid);
        }
    } catch (Exception e) {
        log.error("Exception: " + e);
    }
    return res == 1 ? success : error;
}
```

### 3. Redis 缓存商品库存信息

虽然限流能够过滤掉一些无效的请求，但是还是会有很多请求落在数据库上，通过 `Druid` 监控可以看出，实时查询库存的语句被大量调用，对于每个没有被过滤掉的请求，都会去数据库查询库存来判断库存是否充足，对于这个查询可以放在缓存 Redis 中，Redis 的数据是存放在内存中的，速度快很多。

![](<https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/Redis%20%E7%BC%93%E5%AD%98%E5%BA%93%E5%AD%98%E4%BF%A1%E6%81%AF.png>)

#### 缓存预热

在秒杀开始前，需要将秒杀商品信息提前缓存到 Redis 中，这么秒杀开始时则直接从 Redis 中读取，也就是缓存预热，Springboot 中开发者通过 `implement ApplicationRunner` 来设定 SpringBoot 启动后立即执行的方法

```java
@Component
public class RedisPreheatRunner implements ApplicationRunner {

    @Autowired
    private StockService stockService;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // 从数据库中查询热卖商品，商品 id 为 1
        Stock stock = stockService.getStockById(1);
        // 删除旧缓存
        RedisPoolUtil.del(RedisKeysConstant.STOCK_COUNT + stock.getCount());
        RedisPoolUtil.del(RedisKeysConstant.STOCK_SALE + stock.getSale());
        RedisPoolUtil.del(RedisKeysConstant.STOCK_VERSION + stock.getVersion());
        //缓存预热
        int sid = stock.getId();
        RedisPoolUtil.set(RedisKeysConstant.STOCK_COUNT + sid, String.valueOf(stock.getCount()));
        RedisPoolUtil.set(RedisKeysConstant.STOCK_SALE + sid, String.valueOf(stock.getSale()));
        RedisPoolUtil.set(RedisKeysConstant.STOCK_VERSION + sid, 			String.valueOf(stock.getVersion()));
    }
}
```

#### 缓存和数据一致性

缓存和 DB 的一致性是一个讨论很多的问题，推荐看参考中的 [使用缓存的正确姿势](<https://juejin.im/post/5af5b2c36fb9a07ac65318bd#heading-11>)，首先看下先更新数据库，再更新缓存策略，假设 A、B 两个线程，A 成功更新数据，在要更新缓存时，A 的时间片用完了，B 更新了数据库接着更新了缓存，这是 CPU 再分配给 A，则 A 又更新了缓存，这种情况下缓存中就是脏数据，具体逻辑如下图所示：

![](<https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/%E5%85%88%E6%9B%B4%E6%96%B0%E6%95%B0%E6%8D%AE%E5%BA%93%E5%86%8D%E6%9B%B4%E6%96%B0%E7%BC%93%E5%AD%98.png>)

那么，如果避免这个问题呢？就是缓存不做更新，仅做删除，先更新数据库再删除缓存。对于上面的问题，A 更新了数据库，还没来得及删除缓存，B 又更新了数据库，接着删除了缓存，然后 A 删除了缓存，这样只有下次缓存未命中时，才会从数据库中重建缓存，避免了脏数据。但是，也会有极端情况出现脏数据，A 做查询操作，没有命中缓存，从数据库中查询，但是还没来得及更新缓存，B 就更新了数据库，接着删除了缓存，然后 A 又重建了缓存，这时 A 中的就是脏数据，如下图所示。但是这种极端情况需要数据库的写操作前进入数据库，又晚于写操作删除缓存来更新缓存，发生的概率极其小，不过为了避免这种情况，可以为缓存设置过期时间。

![](<https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/%E5%85%88%E6%9B%B4%E6%96%B0%E6%95%B0%E6%8D%AE%E5%BA%93%E5%86%8D%E5%88%A0%E9%99%A4%E7%BC%93%E5%AD%98.png>)

安装先更新数据库再删除缓存的策略来执行，代码如下所示：

```java
@Override
public int createOrderWithLimitAndRedis(int sid) throws Exception {
    // 校验库存，从 Redis 中获取
    Stock stock = checkStockWithRedis(sid);
    // 乐观锁更新库存和Redis
    saleStockOptimsticWithRedis(stock);
    // 创建订单
    int res = createOrder(stock);
    return res;
}
// Redis 校验库存
private Stock checkStockWithRedisWithDel(int sid) throws Exception {
    Integer count = null;
    Integer sale = null;
    Integer version = null;
    List<String> data = RedisPoolUtil.listGet(RedisKeysConstant.STOCK + sid);
    if (data.size() == 0) {
        // Redis 不存在，先从数据库中获取，再放到 Redis 中
        Stock newStock = stockService.getStockById(sid);
        RedisPoolUtil.listPut(RedisKeysConstant.STOCK + newStock.getId(), String.valueOf(newStock.getCount()),
                              String.valueOf(newStock.getSale()), String.valueOf(newStock.getVersion()));
        count = newStock.getCount();
        sale = newStock.getSale();
        version = newStock.getVersion();
    } else {
        count = Integer.parseInt(data.get(0));
        sale = Integer.parseInt(data.get(1));
        version = Integer.parseInt(data.get(2));
    }
    if (count < 1) {
        log.info("库存不足");
        throw new RuntimeException("库存不足 Redis currentCount: " + sale);
    }
    Stock stock = new Stock();
    stock.setId(sid);
    stock.setCount(count);
    stock.setSale(sale);
    stock.setVersion(version);
    // 此处应该是热更新，但是在数据库中只有一个商品，所以直接赋值
    stock.setName("手机");
    return stock;
}
private void saleStockOptimsticWithRedisWithDel(Stock stock) throws Exception {
    // 乐观锁更新数据库
    int res = stockService.updateStockByOptimistic(stock);
    // 删除缓存，应该使用 Redis 事务
    RedisPoolUtil.del(RedisKeysConstant.STOCK + stock.getId());
    log.info("删除缓存成功");
    if (res == 0) {
        throw new RuntimeException("并发更新库存失败");
    }
}
```

在 Jmeter 压力测试中，并发效果并不好，跟前面的限流并发差不多，观察 Redis 中的数据看出，由于每次都删除缓存，因此导致多次缓存都不能命中，能命中缓存的次数很少，因此这种方案并不可取。

考虑到使用乐观锁更新数据库，因此在使用先更新数据库再更新缓存的策略中，实际情况如下所示

![](<https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/%E5%85%88%E6%9B%B4%E6%96%B0%E6%95%B0%E6%8D%AE%E5%BA%93%E5%86%8D%E6%9B%B4%E6%96%B0%E7%BC%93%E5%AD%98V2.png>)

在 A 未更新缓存阶段，虽然 B 从缓存中获取到的库存信息脏数据，但是，乐观锁使得 B 在更新数据库时失败，这时 A 又更新了缓存，则保证了数据的最终一致性，并且由于缓存一直都可以命中，对并发量的提升也是很显著的。

```java
@Override
public int createOrderWithLimitAndRedis(int sid) throws Exception {
    // 校验库存，从 Redis 中获取
    Stock stock = checkStockWithRedis(sid);
    // 乐观锁更新库存和Redis
    saleStockOptimsticWithRedis(stock);
    // 创建订单
    int res = createOrder(stock);
    return res;
}
// Redis 中校验库存
private Stock checkStockWithRedis(int sid) throws Exception {
    Integer count = Integer.parseInt(RedisPoolUtil.get(RedisKeysConstant.STOCK_COUNT + sid));
    Integer sale = Integer.parseInt(RedisPoolUtil.get(RedisKeysConstant.STOCK_SALE + sid));
    Integer version = Integer.parseInt(RedisPoolUtil.get(RedisKeysConstant.STOCK_VERSION + sid));
    if (count < 1) {
        log.info("库存不足");
        throw new RuntimeException("库存不足 Redis currentCount: " + sale);
    }
    Stock stock = new Stock();
    stock.setId(sid);
    stock.setCount(count);
    stock.setSale(sale);
    stock.setVersion(version);
    // 此处应该是热更新，但是在数据库中只有一个商品，所以直接赋值
    stock.setName("手机");

    return stock;
}
// 更新 DB 和 Redis
private void saleStockOptimsticWithRedis(Stock stock) throws Exception {
    int res = stockService.updateStockByOptimistic(stock);
    if (res == 0){
        throw new RuntimeException("并发更新库存失败") ;
    }
    // 更新 Redis
    StockWithRedis.updateStockWithRedis(stock);
}
// Redis 多个写入操作的事务
public static void updateStockWithRedis(Stock stock) {
    Jedis jedis = null;
    try {
        jedis = RedisPool.getJedis();
        // 开始事务
        Transaction transaction = jedis.multi();
        // 事务操作
        RedisPoolUtil.decr(RedisKeysConstant.STOCK_COUNT + stock.getId());
        RedisPoolUtil.incr(RedisKeysConstant.STOCK_SALE + stock.getId());
        RedisPoolUtil.incr(RedisKeysConstant.STOCK_VERSION + stock.getId());
        // 结束事务
        List<Object> list = transaction.exec();
    } catch (Exception e) {
        log.error("updateStock 获取 Jedis 实例失败：", e);
    } finally {
        RedisPool.jedisPoolClose(jedis);
    }
}
```

### 4. Kafka 异步

##  数据库建表

```mysql
CREATE TABLE `stock` (
    `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
    `name` varchar(50) NOT NULL DEFAULT '' COMMENT '名称',
    `count` int(11) NOT NULL COMMENT '库存',
    `sale` int(11) NOT NULL COMMENT '已售',
    `version` int(11) NOT NULL COMMENT '乐观锁，版本号',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;
CREATE TABLE `stock_order` (
    `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
    `sid` int(11) NOT NULL COMMENT '库存ID',
    `name` varchar(30) NOT NULL DEFAULT '' COMMENT '商品名称',
    `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=55 DEFAULT CHARSET=utf8;
```

## 参考

>- [极客时间：许令波 - 如何设计一个秒杀系统](<https://time.geekbang.org/column/intro/127>)
>- [crossoverjie：SSM(十八)秒杀架构实践](<https://crossoverjie.top/2018/05/07/ssm/SSM18-seconds-kill/>)
>- [秒杀系统优化方案（下）吐血整理](<https://www.cnblogs.com/xiangkejin/p/9351501.html>)
>- [电商网站秒杀与抢购的系统架构](http://www.codeceo.com/article/spike-system-artch.html)
>- [使用缓存的正确姿势](<https://juejin.im/post/5af5b2c36fb9a07ac65318bd#heading-11>)