# Redis

# 数据类型及适用场景

Redis 主要有一下集中数据类型：

- String
- Hash
- List
- Set
- Sorted Set

> Redis 除了这 5 种数据类型之外，还有 Bitmaps、HyperLogLogs、Streams 等。

## String 字符串

**存储类型**

这是最简单的类型，存储字符串、整数、浮点数。

**操作命令**

```bash
set key value [EX seconds] [PX milliseconds] [NX|XX]
get key
```

**使用场景**

- 热点数据、静态低改动数据的缓存
- 分布式锁服务
- 分布式 session
- 全局 id 使用的时候取一个批次的数据（INCR 方法）
- 限流，设置一个带有过期时间的key，value存限流次数，通过在过期时间内判断value来实现 

## Hash 哈希

**存储类型**

包含键值对的无序散列表。一个 key 下有多个键值对

**操作命令**

```bash
hset website baidu www.baidu.com
hget website taobao www.taobao.com
hexists website baidu
```

**使用场景**

- 类似购物车的多个商品信息

## List

**存储类型**

存储有序的字符串，元素可以重复。可以充电队列和栈的角色。

**操作命令**

```bash
lpush key a 
rpush key b
lpop key
rpop key
lindex key 0	# 获取第0位的value
lrange key 0 -1 # 获取范围内所有内容
```

**使用场景**

- 因为是有序的，可以用来做类似朋友圈的时间线
- 队列（先进先出）
- 栈（先进后出）

## Set

**存储类型**

string 类型无序集合

**操作命令**

```bash
# 单个 set 的操作
sadd key a b c d e # 添加元素
smembers key # 获取所有元素
scard key # 统计元素个数
srandmember key # 随机获取一个元素
spop key # 随机弹出一个元素
srem key a # 移除元素
sismember key a # 查看元素是否存在

# 多个 set 操作
# 获取差集
sdiff set1 set2
# 获取交集
sinter set1 set2
# 获取并集
sunion set1 set2
```

**使用场景**

- 抽奖（随机弹出一个抽奖用户）
- 点赞（朋友圈id 作为key， 维护所有点赞用户；sadd点赞；srem取消点赞；smembers点赞用户；scard 点赞数）
- 打卡（类似点赞）
- 商品标签、筛选（将标签作为key，商品作为value，通过求多个set并集返回商品筛选信息）
- 用户关注、推荐模型

## Sorted Set （zset）

**存储类型**

有序的set集合，每个元素有个 scope

scope 相同时，按照key的ascii码排序

**操作命令**

```bash
zadd key 10 java 20 js 30 php 40 c # 添加元素
zincrby key 15 java
zrange key 0 -1 withscores # 获取全部元素
zrangebyscore key 20 30 # 获取指定score返回的元素
zrevrange key 0 10 withscores # 倒序获取0-10个元素
zrem key php # 删除元素
zcard key # 统计元素个数
zcount key 20 40 # 统计分值内的个数
zrank key java # 获取元素的rank
zscore key java # 获取元素的score
```

**使用场景**

- 热点新闻排行榜（zincrby：增加点击量，zrevrange 获取点击最多的新闻）

## bitmap

字符串上的位操作。统计用于在线情况，活跃用户等。

## Hyperloglos

提供了一种不太准确的基数统计方法，比如统计网站的UV、PV，存在一定误差。

## Sreams

支持多播的可持久化消息队列，用于实现发布订阅功能。



# 原理解析

## 事务



## 运行机制





## 内存回收机制





## 持久化







# 集群



# 实战

