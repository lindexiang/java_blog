[TOC]
# 1. 单机数据库的实现

# 1. 简介 

redis客户端默认目标数据库为0号数据库，可以通过SELECT命令来切换目标数据库。默认是存在16个。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200703210648.png)

## 1.1 数据库的键空间

Redis是一个键值对数据库服务器，服务器中的每个数据库都由一个redisDb结构表示，其中redisDB的dict字典保存了数据库中的所有键值对，我们将这个字典称为键空间。

```c
// 数据库的键空间
typedef struct redisDb{
    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict
} redisDb;
//客户端
typedef struct redisClient{
    //记录客户端当前正在使用的数据库
    redisDb *db;
} redisClient;
```

键空间和用户所见的数据库是直接对应的

1. 键空间的键也就是数据库的键，每个键都是一个字符串对象。
2. 键空间的值也就是数据库的值，每个值可以是字符串对象，列表对象，哈希表对象，集合对象和有序集合对象中任意一种Redis对

示意图如下所示

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200703212045.png)



## 1.2 设置键的过期时间

redis也是使用一个dict来保存每个键的过期时间

```c
typedef struct redisDb{
    //过期字典，保存着键的过期时间
    dict *expires;
} redisDb;
```

所以一个redisDb里存在2个dict一个保存数据，一个是保存键的过期时间。示意图如下所示

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200703212402.png)

## 1.3 过期键删除策略

对于过期键的删除策略一般如下几种，单独使用一种删除策略都会有明显的缺陷

- 定时删除
  定时删除会占用大量的CPU，影响服务器的响应时间和吞吐量。

- 惰性删除

  过期键会大量存在内存中，浪费内存且有内存泄漏的风险

- 定期删除
  定期删除是每隔一段时间进行删除，限制删除的次数

redis为了保证服务器的响应性能，采用的删除策略是利用**惰性删除+定期删除策略**。具体的步骤如下

- 定期删除 
  redis默认是每隔100ms会随机抽泣一些设置了过期时间的key，检查是否过期，如果过期了就删除。**注意点:不是选择所有的key遍历，会造成CPU过高**
- 惰性删除
  每次获取key时候，会检查这个key是否设置了过期时间，如果设置了过期时间并且已经过期就会删除并且不返回任何东西

# 2 redis的数据淘汰机制

**redis使用惰性删除+定期删除策略还是会存在很多过期的key。这时候如果内存不足就要走内存淘汰机制了**。为了更好的利用内存，使redis内存中缓存的都是热点数据，Redis设计了相应的内存淘汰机制。通过设置**maxmemory**可以配置允许用户使用的最大内存大小。当数据集的大小到达一定的大小，就会根据内存淘汰策略进行数据淘汰。

## 2.1 redis的6种内存淘汰策略

redis的内存淘汰过程如下

1. 客户端发起set等申请内存的操作
2. redis会检查内存的使用情况，如果已使用的内存大于maxmemory配置的值，那么就会根据用户配置的不同淘汰策略来淘汰内存
3. 如果执行成功，就会执行该命令。

6种内存淘汰策略

- no-eviction策略 这是redis默认的 不淘汰内存 写操作会返回错误
- volatile-lru策略 在设置了过期时间的数据集server.db[i].expires中使用LRU策略淘汰数据
- allkeys-lru 策略 从当前数据集中使用lru策略淘汰数据 server.db[i].dcit
- volatile-ttl策略 从已设置过期时间的数据集中淘汰即将过期的数据
- volatile-random 在设置过期时间的数据集中随机淘汰
- allkeys-random 在数据集中随机淘汰

**redis在确定删除某一个键后，会在删除这个键后并发布数据变更消息发布到本地(AOF持久化)和主从(主从链接)**

redis实现的是近似的LRU淘汰机制，是会随机取出几个元素，并判断该key的上次访问时间，并删除最早访问的元素。

## 2.2 LRU算法和LRU-K算法

LRU算法是least recently used **距离现在最久的数据**。用一个双端链表+Map可以实现 set和get的时间复杂度是O(1)  具体步骤如下

1. get和set时先从map表中读取key是否存在。如果不存在在list尾部插入key，map中put
2. 如果key存在，就在list中找到这个key，先删除，然后在tail插入这个元素。
3. 淘汰key，就从head处删除一个key

使用LRU算法会有一个问题

**LRU的缓存污染。当进行一个批量性的操作可能会将数据库中原有的热点数据都删除，导致缓存中的数据都不是热点数据。**

改进方案 使用LRU-K算法

LRU-K指的是在**K次访问距离现在最久的淘汰**

多加一个链表+Map来保存每个key的访问次数 只有在这个里面访问超过K次才能被加入上面，这样就可以保证第一个链表中都是热点数据。

# 3 redis的持久化机制

**redis是一个键值对内存数据库，数据存储在内存当中。在处理客户端操作时，所有的操作都是在内存当中进行。为了防止数据的丢失和重启的数据快速恢复，redis的持久化是非常重要的**。为了避免数据在内存当中丢失，redis提供了对持久化的支持，可以采用不同的方式将数据保存下来。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200705230653.png)

## 3.1 RDB持久化机制

**RDB是快照存储持久化方式，将redis某一刻的内存数据保存在硬盘当中。在redis服务器启动后，会重新加载文件到内存当中**。开启持久化的方式有2种

1. 客户端往服务端发送save指令
   save是一个同步操作，服务端会阻塞在save命令，之后客户端其他请求都无法响应直到数据同步完成。
2. 客户端往服务端发送bgsave指令
   客户端往服务端发送bgsave指令后，服务端主进程会forks一个子进程来同步数据，在子进程同步完数据后会退出

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200705232012.png)

和save指令相比，redis服务器在处理bgsave时采用子线程对io写入，而主线程可以接收其他请求。forks子进程的过程中一样不能接收其他请求。一般情况下是很快的。

**配置文件配置rdb持久化**

```c
# 900s内至少达到一条写命令
save 900 1
# 300s内至少达至10条写命令
save 300 10
# 60s内至少达到10000条写命令
save 60 10000
```

**即在达到触发条件后就会开启bgsave，缺点是时间配置太短会频繁触发影响性能，太长容易造成数据丢失。**

## 3.1.1 RDB的优点和缺点

RDB的优点

1. 和AOF方式相比，rdb文件恢复的速度快
2. rdb文件非常紧凑，适合备份数据
3. 通过rdb进行数据备份，fork子进程进行，对服务器的性能影响较小。

RDB的缺点

1. 服务器宕机的话，RDB方式会造成某个时间段数据的丢失，比如设置5分钟同步数据，那么这段时间内的数据容易丢失。
2. 使用save指令会造成服务器堵塞。使用bgsave会forks子进程，如果数据量太大，fork的过程会阻塞，并且子进程会额外消耗内存。

## 3.2 AOF持久化机制

AOF持久化机制(append-only file)和RDB快照模式不同，会持久化客户端对服务端的每一次写操作，并且将写操作追加的方式保存到aof文件的结尾。在redis服务器重启后，会加载并运行aof文件达到恢复数据的方式。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200705235223.png)

AOF持久化机制也是需要在配置文件中开启。存在三种的写入策略。

1. always 每一个写操作都会保存到aof文件中，这种操作很安全，但是每一个写操作都是一次IO，会很慢。
2. everysec 是默认的机制，每秒写入一次，最多丢失1s的数据。
3. no redis服务器不负责写入aof文件的操作，由操作系统决定何时写入aof文件，更快但是不安全。

### 3.2.1 AOF 文件压缩和重写

客户端的每一次操作都会追加到AOF文件的末尾，那么AOF文件会非常大。在恢复的时候速度会很慢。为了解决这个问题，redis支持aof文件的重写和压缩，将多条重复指令合并成一条。

重写的指令是bgrewriteaof。redis主进程也会forks一个子进程来处理。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200705235759.png)

aof文件重写的好处是

1. 减少文件的大小，减少磁盘占用量。
2. 将aof命令压缩为最小命令集，加快数据恢复的速度。

### 3.2.2 AOF的优点和缺点

优点：

1. AOF只是追加日志文件，对服务器的性能影响小，相比与RDB速度要快，对内存占用小。
2. AOF文件的数据更加完整相比于RDB而言

缺点：

1. AOF生成的日志文件较大，通过AOF重写的方式，文件体积很大。
2. 恢复数据的速度比RDB慢

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200706000210.png)

# 4. redis的事件机制

redis采用的是事件驱动机制来处理大量的网络IO。redis 的事件驱动只关注2种类型的事件

1. 文件事件 即 file event 服务端和客户端之间的socket链接
2. 事件事件 time event redis的定时器执行  比如定时执行或者延时执行的事件

## 4.1 redis的速度快原因

1. redis的大部分请求都是纯内存操作
2. 采用单线程，避免了不必要的上下文切换和竞争条件
3. 非阻塞IO 多路IO服用

## 4.2 redis的事件机制原理

### 4.2.1 文件事件

redis的事件驱动的示意图如下所示，主要分为1. socket套接字，I/O多路服用、文件事件分发(file event dispather)、事件处理器。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200706220328.png)



**每一个socket 链接对应到linux内核都会创建对应的一个file discription**，当socket准备好accept、write、read、close等操作时，都会触发一个事件。所以redis可能同时会并发出现多个事件。

多个文件事件并发出现后，redis的I/O多路服用事件会放在同一个就绪事件队列里。然后事件分发器会有序、同步的处理该事件

### 4.2.2 时间事件

**redis的时间事件分为1. 定时事件和2.周期性事件**

redis会定义一个时间事件的结构体。并将这个结构体放在一个无序的链表中。

1. 定时事件 当事件被触发后会从链表中移除

2. 周期性事件 当事件触发后会更新下一次触发的时间

## 4.3 redis的事件处理

redis同时会存在时间事件和文件事件，所以服务器必须对2个事件进行调度，决定何时处理文件事件、何时处理时间事件。

redis在main函数里会while无限循环的方式调用函数处理所有的事件。

1. 计算距离当前最近的时间事件，计算一个超时的时间
2. 调用epoll()函数等待底层的I/O复用事件就绪或者超时被中断、
3. 当epoll()函数返回后会处理所有的文件事件和已经到达的时间事件

伪代码如下

```c
/* 伪代码 */
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    /* 获取到达时间距离当前时间最接近的时间事件*/
    time_event = aeSearchNearestTimer();
    /* 计算最接近的时间事件距离到达还有多少毫秒*/
    remaind_ms = time_event.when - unix_ts_now();
    /* 如果事件已经到达，那么remaind_ms为负数，将其设置为0 */
    if (remaind_ms < 0) remaind_ms = 0;
    /* 根据 remaind_ms 的值，创建 timeval 结构*/
    timeval = create_timeval_with_ms(remaind_ms);
    /* 阻塞并等待文件事件产生，最大阻塞时间由传入的 timeval 结构决定，如果remaind_ms 的值为0，则aeApiPoll 调用后立刻返回，不阻塞*/
    /* aeApiPoll调用epoll_wait函数，等待I/O事件*/
    aeApiPoll(timeval);
    /* 处理所有已经产生的文件事件*/
    processFileEvents();
    /* 处理所有已经到达的时间事件*/
    processTimeEvents();
}
```

## 4.4 删除事件

当不需要某一个fd事件时，redis就会把该fd删除，步骤如下

1. 根据fd在未就绪表中查找到事件
2. 将对应的事件的标记取消
3. 将该事件从内核中对应的事件监听取消。
