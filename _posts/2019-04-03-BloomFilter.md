---
layout: post
title:  "浅析布隆过滤器"
categories: [分布式, 缓存, 架构, BloomFilter]
tags:  分布式 Redis 缓存 BloomFilter
author: G.Fukang
---
浅析布隆过滤器原理以及 Java 实现

# 布隆过滤器原理

布隆过滤器是用来判断一个元素是否出现在给定集合中的重要工具，具有快速，比哈希表更节省空间等优点，而缺点是存在一定的误识别率（fast-positive），也就是它可能会把不是集合内的元素判定为存在于集合内，不过这样的概率相当小。

其原理也比较简单，如图所示，S 集合中有 n 个元素，利用 k 个哈希函数，将 S 中的每个元素映射到一个**长度为 m 的**位(bit) 数组 B 中的不同位置上，这些位置上的二进制数均置为 1，如果待检测的元素经过这 **k 个哈希函数**的映射后，发现其 k 个位置哈希的二进制数不全是 1，那么这个元素一定不在集合 S 中，反之该元素可能是 S 中的某一个元素。

![](<https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/BloomFilter.jpg>)

根据前面的描述可以看到，**为了估算出需要多少个哈希函数，已经创建长度为多少的 bit 数组合适，在构造一个布隆过滤器时，需要传入两个参数，即可以接收的误判率 fpp 和元素总个数 m**

关于误差的计算，可以按照一下的方法：

假定布隆过滤器有 m 比特，里面有 n 个元素，每个元素对应 k 个信息指纹的 hash 函数，在这个布隆过滤器插入一个元素，那么比特位被设置成 1 的概率为 `1/m`，它依然为 0 的概率为 `1 - 1/m`，那么 k 个哈希函数都没有把他设置成 1 的概率为 `(1-1/m)^k` ，一个比特在插入了 n 个元素后，被设置为 1 的概率为 `1 - (1/m)^kn`

# 使用 Guava 中的 BloomFilter

使用 Guava，需要先在 pom.xml 文件中引入以下依赖：

```java
<dependency>
	<groupdId>com.google.guava</groupId>
	<artifactId>guava</artifactId>
	<version>19.0</version>
</dependency>
```

测试程序如下

```java
public class GuavaBloomFilterTest {
	// BloomFilter 容量
    private static final int capacity = 10000000;

    private static final int key = 9999998;
	// 构建 BloomFilter
    private static BloomFilter<Integer> bloomFilter = BloomFilter.create(Funnels.integerFunnel(), capacity);
	// 填充数据
    static {
        for (int i = 0; i < capacity; i++) {
            bloomFilter.put(i);
        }
    }

    public static void main(String[] args) {
        long startTime = System.nanoTime();

        if (bloomFilter.mightContain(key)) {
            System.out.println( " key : " + key + " 包含在布隆过滤器中 ");
        }

        long endTime = System.nanoTime();

        System.out.println("消耗时间: " + (endTime - startTime) + " 微秒");

        // 错误率判断
        double errNums = 0;
        for (int i = capacity + 1000000; i < capacity + 2000000; i++) {
            if (bloomFilter.mightContain(i)) {
                ++errNums;
            }
        }

        System.out.println("错误率: " + (errNums/1000000));
    }
}
/** output **/
key : 9999998 包含在布隆过滤器中 
消耗时间: 304000 微秒
错误率: 0.029828
```

可以看到，误判率在 `0.03` 左右，Guava 也给定了显示指定误判率的接口，跟踪断点可以看到，误判率为 `0.02` 时数组的大小为 8142363，误判率为 `0.03` 时数组大小为 7298440，可以看出误判率降低了 `0.01` 其底层数组的大小也减小了 843923

# Java 实现布隆过滤器

首先建立一个 bit 数组，将其值均置为 0，添加元素时使用乘法哈希，利用 n 个不同的哈希函数计算出 n 个不同的信息指纹（在 bit 数组中的索引），然后将这 n 个信息指纹在 bit 数组中对应置为 1，这样就构成了一个布隆过滤器。检索元素时是同样的道理，使用相同的算法得出 n 个信息指纹，如果这 n 个信息指纹在 bit 数组对应位置均为 1，则表示这个元素存在。这部分只是简单实现，所以指定了哈希函数的个数和 bit 数组的长度

**通用接口**

```java
public interface BloomFilter {
    // 添加元素
    public void put(String key);
    // 判断元素是否存在
    public boolean mightContain(String key);
}
```

**乘法 Hash 算法**

```java
public class Hash {
    // bit 数组容量
    private int capacity;
    // 哈希因子
    private int seed;

    public Hash(int capacity, int seed) {
        this.capacity = capacity;
        this.seed = seed;
    }

    // 乘法 Hash 算法
    public int Hash(String key) {
        int res = 0;
        int length = key.length();

        for (int i = 0; i < length; i++) {
            res = seed * res + key.charAt(i);
        }
        // 将结果限定在 capacity 内
        return (capacity - 1) & res;
    }
}
```

**实现**

```java
public class SimpleBloomFilter implements BloomFilter {

    // bit 数组的长度
    private static final int DEFAULT_SIZE = 2 << 24;
    // Hash 算法中所用的乘数
    private static final int[] seeds = new int[]{3, 5, 7, 11, 13, 17, 19, 23};
    // bit 数组
    private static BitSet bitSet = new BitSet(DEFAULT_SIZE);
    // Hash 函数
    private static Hash[] hashes = new Hash[seeds.length];

    @Override
    public void put(String key) {
        if (key != null) {
            // 将 key Hash 为 8 个或者多个整数，然后在这些整数的 bit 上变为 1
            for (Hash hash : hashes) {
                bitSet.set(hash.Hash(key), true);
            }
        }
    }

    @Override
    public boolean mightContain(String key) {
        if (key == null) {
            return false;
        }
        boolean res = true;
        for (Hash hash : hashes) {
            res = res && bitSet.get(hash.Hash(key));
            if (!res) {
                return res;
            }
        }
        return res;
    }

    public Hash[] build() {
        Hash[] res = new Hash[seeds.length];
        for (int i = 0; i < seeds.length; i++) {
            res[i] = new Hash(DEFAULT_SIZE, seeds[i]);
        }
        hashes = res;
        return hashes;
    }
}
```

**测试代码**

```java
public class SimpleBloomFilterTest {

    private static final Logger LOGGER = LoggerFactory.getLogger(SimpleBloomFilterTest.class);

    private static SimpleBloomFilter bloomFilter = new SimpleBloomFilter();

    static {
        bloomFilter.build();

        for (int i = 0; i < 2000000; i++) {
            bloomFilter.put(String.valueOf(i));
        }
    }

    public static void main(String[] args) {

        String testKey = "9999";

        long startTime = System.nanoTime();

        if (bloomFilter.mightContain(testKey)) {
            LOGGER.info(" key : " + testKey + " 包含在布隆过滤器中 ");
        }

        long endTime = System.nanoTime();

        LOGGER.info("消耗时间: " + (endTime - startTime) + " 微秒");

        double errNum = 0;
        for (int i = 2000000; i < 4000000; i++) {
            if (bloomFilter.mightContain(String.valueOf(i))) {
                ++errNum;
            }
        }

        LOGGER.info("误差：" + (errNum / 2000000) * 100 + "%");
    }
}
/** output **/
key : 9999 包含在布隆过滤器中
消耗时间: 15311602 微秒
误差：0.01415%
```

# Redission 中的分布式布隆过滤器

不论是在 Guava 中，还是自己的简单实现，都只是本地的布隆过滤器，仅仅存在单个应用中，同步起来十分复杂，而且一旦应用重启，则之前添加的元素均丢失，对于分布式环境，可以利用 Redis 构建分布式布隆过滤器

Redisson 框架提供了布隆过滤器的实现

```java
RBloomFilter<SomeObject> bloomFilter = redisson.getBloomFilter("sample");
// 初始化布隆过滤器，预计统计元素数量为55000000，期望误差率为0.03
bloomFilter.tryInit(55000000L, 0.03);
bloomFilter.add(new SomeObject("field1Value", "field2Value"));
bloomFilter.add(new SomeObject("field5Value", "field8Value"));
bloomFilter.contains(new SomeObject("field1Value", "field8Value"));
```

简单分析下 Redission 中源码

**通用接口**

```java
public interface RBloomFilter<T> extends RExpirable {
    boolean add(T object)
    boolean contains(T object)
    boolean tryInit(long expectedInsertions, double falseProbability);
}
```

**接口实现**

```java
public class RedissonBloomFilter<T> extends RedissonExpirable implements RBloomFilter<T> {

    public boolean add(T object) {
        // 构造多个哈希值
        long[] hashes = hash(object);

        while (true) {
            if (size == 0) {
                // 配置以哈希表的形式存在 Redis 中
                // 这里执行 HGETALL
                readConfig();
            }

            int hashIterations = this.hashIterations;
            long size = this.size;

            // 需要设置为 1 的索引
            long[] indexes = hash(hashes[0], hashes[1], hashIterations, size);

            // 省略部分代码 ${新建客户端}

            // 依次执行 set 操作
            for (int i = 0; i < indexes.length; i++) {
                bs.setAsync(indexes[i]);
            }
            try {
                List<Boolean> result = (List<Boolean>) executorService.execute();

                for (Boolean val : result.subList(1, result.size()-1)) {
                    if (!val) {
                        return true;
                    }
                }
                return false;
            } catch (RedisException e) {
                if (!e.getMessage().contains("Bloom filter config has been changed")) {
                    throw e;
                }
            }
        }
    }
}
```

GET 函数这里就不再深入探讨，只是将 add 函数中的 SET 变成 GET 操作

**Hash 函数**

```java
private long[] hash(Object object) {
    ByteBuf state = encode(object);
    try {
        return Hash.hash128(state);
    } finally {
        state.release();
    }
}
```

这个函数将 Object 编码后，返回一个 Byte 数组，然后调用 `Hash.hash128` 计算哈希值，这里的哈希算法是 HighwayHash

```java
public static long[] hash128(ByteBuf objectState) {
    HighwayHash h = calcHash(objectState);
    return h.finalize128();
}

protected static HighwayHash calcHash(ByteBuf objectState) {
    HighwayHash h = new HighwayHash(KEY);
    int i;
    int length = objectState.readableBytes();
    int offset = objectState.readerIndex();
    byte[] data = new byte[32];

    // 分区计算哈希
    for (i = 0; i + 32 <= length; i += 32) {
        objectState.getBytes(offset  + i, data);
        h.updatePacket(data, 0);
    }
    if ((length & 31) != 0) {
        data = new byte[length & 31];
        objectState.getBytes(offset  + i, data);
        h.updateRemainder(data, 0, length & 31);
    }
    return h;
}

// 第二个哈希函数，计算最后的索引值
private long[] hash(long hash1, long hash2, int iterations, long size) {
    long[] indexes = new long[iterations];
    long hash = hash1;

    // 多次迭代
    for (int i = 0; i < iterations; i++) {
        indexes[i] = (hash & Long.MAX_VALUE) % size;

        // 根据迭代次数选择哈希值，累加
        if (i % 2 == 0) {
            hash += hash2;
        } else {
            hash += hash1;
        }
    }
    return indexes;
}
```

# 布隆过滤器解决缓存穿透问题

关于缓存穿透问题可以在之前写的博客[如何应对缓存问题](<https://gongfukangee.github.io/2019/04/02/Cache/#%E7%BC%93%E5%AD%98%E7%A9%BF%E9%80%8F>)查看。解决缓存穿透问题可以使用缓存空对象和布隆过滤器两种方法，这里仅讨论布隆过滤器方法。

使用布隆过滤器逻辑如下：

1. 根据 key 查询缓存，如果存在对应的值，直接返回；如果不存在则继续执行
2. 根据 key 查询缓存在布隆过滤器的值，如果存在值，则说明该 key 不存在对应的值，直接返回空，如果不存在值，继续向下执行
3. 查询 DB 对应的值，如果存在，则更新到缓存，并返回该值，如果不存在值，则更新到布隆过滤器中，并返回空

具体流程图如下所示：

![](<https://github.com/gongfukangEE/gongfukangEE.github.io/raw/master/_pic/%E5%88%86%E5%B8%83%E5%BC%8F/%E7%BC%93%E5%AD%98%E7%A9%BF%E9%80%8F.png>)

代码实现，完整的代码可以到[GitHub](https://github.com/gongfukangEE/my-distributed-tools)查看

```java
public String getByKey(String key) {
    String value = get(key);
    if (StringUtils.isEmpty(value)) {
        logger.info("Redis 没命中 {}", key);
        if (bloomFilter.mightContain(key)) {
            logger.info("BloomFilter 命中 {}", key);
            return value;
        } else {
            if (mapDB.containsKey(key)) {
                logger.info("更新 Key {} 到 Redis", key);
                String valDB = mapDB.get(key);
                set(key, valDB);
                return valDB;
            } else {
                logger.info("更新 Key {} 到 BloomFilter", key);
                bloomFilter.put(key);
                return value;
            }
        }
    } else {
        logger.info("Redis 命中 {}", key);
        return value;
    }
}
```

# 参考

- [Redis 布隆过滤器实战「缓存击穿、雪崩效应」](https://juejin.im/post/5c9442ae5188252d77392241)
- [Redis缓存穿透、缓存雪崩、redis并发问题分析](https://juejin.im/post/5b961172f265da0ab7198f4d)
- [Redisson 的布隆过滤器实现](https://hellokangning.github.io/zh/post/redisson-bloomfilter-impl/)