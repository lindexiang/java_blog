[TOC]

# 1. Redis的主备切换

redis的读写分离或者为了容灾，需要使用主从模式，一般采用一主多从或者级联的模式，如下图所示。

**那就需要主从复制，一般分为全量同步和增量同步**。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200707001230.png)

## 1.1 redis的主从同步模式

### 1.1.1 全量同步

Redis的全量同步一般发生在slave的初始化。在老版本的redis，slave和master断开重连也会全量同步。具体步骤如下： 

- 从服务器连接主服务器，发送SYNC命令； 
- 主服务器接收到SYNC命令后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令； 
- 主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令； 
- 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照(先将RDB保存到磁盘(非阻塞)，在将RDB的数据恢复到内存中 (阻塞))； 
- 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令； 
- 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；

### 1.1.2 增量同步

redis增量同步是slave初始化之后主服务器的写操作同步到从服务器的过程。

增量同步的过程主服务器每次执行一个写命令都会向从服务器发送相同的写命令，从服务器收到并执行写命令。

## 1.2 Redis的主从同步策略

在redis主从刚刚连接时，进行全量同步，在全量同步结束后就会进行增量同步。同时，redis从服务器在任何时候都可以发送SYNC发起全量同步。

**redis的同步策略是都会先进行增量同步，在同步失败后才会进行全量同步。

## 1.3 Redis主从同步的特点

1. 采用的是异步同步方式
2. 一个master可以存在多个slave。**多个slave会占用master的带宽，影响master的吞吐性能。**slave也可以接受其他slave的链接。
3. 主从复制对master来说是非阻塞的，即在slave在主从同步的过程中master也可以接收客户端的请求。
4. 主从复制对slave来说也是非阻塞的。slave在主从同步时也是可以接收外界的查询请求。只是查询返回的是老的数据。你可以设置此时是拒绝服务。但是slave将同步过来的RDB文件加载到内存的过程是阻塞的。在大数据集时，加载到内存中也是非常耗时。
5. **主从复制提高了redis服务的扩展性，避免单个redis的读写访问压力过大。同时也可以为HA提供解决方案。但是主从复制无法解决数据量过大的问题。要采用集群分片的思路。**
6. **为了提供master的并发，可以配置让master不将数据持久化到磁盘中。采用主从同步的方式来冗余数据。这时要注意的是设置master不能宕机重启。因为会导致master和slave的数据都被清空。**

## 1.4 redis的主从同步如何实现

- 全量同步：

master会forks子进程(forks的过程是阻塞的)用于将redis中的数据生成一个rdb文件，与此同时，master会缓存所有接收到的来自客户端的写命令（包含增、删、改），当子进程处理完毕后，会将该rdb文件传递给slave服务器，而slave服务器会将rdb文件保存在磁盘并通过读取该文件将数据加载到内存，在此之后master服务器会将在此期间缓存的命令发送给slave服务器，然后slave服务器将这些命令依次作用于自己本地的数据集上最终达到数据的一致性。

- 增量同步:

**从redis 2.8版本以前，并不支持部分同步，当主从服务器之间的连接断掉之后，master服务器和slave服务器之间都是进行全量数据同步，**但是从redis 2.8开始支持了部分同步。

部分同步的实现master内存中维护了数据的最大offset和每个slave的同步offset，每个slave服务器在跟master服务器进行同步时都会携带自己的标识和同步offset。当主从连接断掉之后，slave服务器隔断时间（默认1s）主动尝试和master服务器进行连接，如何当前的offset和offset_max差距不大就从偏移量开始同步。否则开启一次全量同步。在增量同步过程中，master会将本地记录的同步备份日志中记录的指令依次发送给slave服务器从而达到数据一致。

## 1.5 redis如何配置主从和主从延迟过大处理

redis的主从同步配置只要使用slaveof命令即可。

```c
slaveof 192.168.1.1 6379
```

当redis的主从延迟严重时就要考虑拒绝master的拒绝写入。因为这时发生主备切换会发生数据丢失。主从延迟检测如下

1. slave每秒会ping master，确认当前的主从同步的进度。
2. master会存储每个slave的最近ping的时间。
3. 用户可以配置N个slave小于M秒的确认延迟，master才可以接受写入操作来保证数据的CAP模型。

## 1.6 redis主从复制数据丢失处理

master和slave之间的数据复制是异步的，MYSQL的读写分离也是异步的。那么有可能在数据没有复制到slave，master就挂了。那么这部分的数据就会丢失。

还有一种情况是脑裂导致的。由于网络不通导致出现了2个master。在这期间的数据就会丢失。

可以增加如下的配置

```c
min-slaves-to-write 1 # 要求至少一个slave
min-slaves-max-lag 10 # 数据复制和同步的延迟不能超过10s
```

1. 异步延迟
   配置min-slaves-max-lag，延迟过大 客户单拒绝写入

2. 脑裂数据丢失

   配置min-slaves-to-write 让没有slave的master拒绝写入数据



![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200711133239.png)



# 2. redis的哨兵模式

在redis的主从复制模式，一旦master发生故障不能提供服务，需要手动将slave切换成master，还要通知客户端更新新的master地址。这在高并发情况下是无法接受的。所以redis提供了哨兵机制来解决该问题。

## 2.1 redis的高可用介绍

高可用指的是当服务发生故障后在多久内能提供正常服务。在redis层面上，高可用要保证能正常提供服务外，还要保证服务的扩展性和数据的安全等。

redis在支持高可用的方式有 **持久化、主从同步、哨兵机制和集群模式**。分别解决了如下问题

1. 持久化 提供了数据的本地备份，保证了在单机节点redis进程重启后数据不会丢失。
2. 主从同步  主从同步是redis高可用的基础，哨兵和集群都是在该基础上实现的。主从同步实现了数据的多机备份、读操作的负载均衡和简单的故障恢复。缺点是无法自动故障切换，写操作无法负载均衡，数据收到单机节点的限制
3. 哨兵机制 哨兵时在主从同步的基础上实现了自动化的故障恢复，缺点和主从同步一样。
4. 集群模式 redis的集群模式解决了写操作无法负载均衡和存储能力受到单机的限制，实现了完善的高可用方案。

## 2.2 redis的Sentinel的架构

Sentinel的架构图如下所示。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200708223928.png)

**主要功能包括了 master的存活检测、主从运行状态检测、自动故障转移(failover)、主从切换。redis的主备配置至少是一主一从**。

Sentinel集群可以用来管理多个Redis服务器。

1. 监控  sentinel会监控master和slave的运行状态。
2. 通知 当sentinel监控某个redis出现问题，就会向集群的其他节点发送通知。
3. 自动故障转移 当master不能提供服务，sentinel会开始一次自动的故障转移工作，将一个slave升级为新的master，并且将其他的slave指向新的master。
4. 提供集群信息  在哨兵模式下，客户端初始化是和sentinel集群链接来获取master的信息。在故障转移后由哨兵更新新的master信息。

### 2.2.1 主观下线和客观下线

在默认情况下，每个sentinel节点会**每秒一次**的频率向所有的redis节点和其他的sentinel节点发送PING命令，并通过回复来判断节点是否在线。

- 主观下线

  主观下线适用master和slave。如果在PING完后`down-atfer-milliseconds`时间内没有收到目标节点的回复，就判断该节点**主观下线**。

- 客观下线

  客观下线适用master。当检测到master发生故障，sentinel会向集群的其他节点发送该节点状态。当超过`quorum`个数的节点判断master故障，那么sentinel就会判断master**客观下线**。

Sentinel的交互命令有一下几个

1. PING 

   Sentinel向redis发送PING指令来获取节点的运行状态

2. INFO

   Sentinel向Redis节点发送INFO指令来获取其从节点信息

3. PUBLISH/SUBSCRIBE 

   Sentinel通过pub来发布自己的信息和主从节点相关的配置。Sentinel通过sub来订阅监控相同集群的Sentinel信息。

## 2.3 领导者哨兵选举流程

当redis主节点下线后，哨兵会选出领导者哨兵来执行下一步的故障转移，哨兵的选举机制为raft算法。每次做故障转移只能由一个Sentinel来控制，所以必须要选出一个master来领导。所以要超过N/2+1的票。
规则：

1. 所有的哨兵都有被选举权。
2. 每次哨兵选举后，不管成功与失败，都会计数epoch。在一个epoch内，所有的哨兵都有一次将某哨兵设置为局部领头的机会，并且局部领头一旦设置，在这个epoch内就不能改变。
3. 每个发现主服务器进入客观下线的哨兵都会要求其他哨兵选举自己。
4. 哨兵选举局部领头的原则是先到先得，后来的拒绝。
5. 如果一个哨兵获得的选票大于半数，则成为领头。



## 2.4 Redis的哨兵模式搭建

一个完善的Sentinel集群至少使用三个Sentinel实例，并且保证实例放在不同的物理机上。

比如搭建一个master，两个slave的主从，3个节点的sentinel集群如下所示。

- 启动redis

拷贝3份redis.conf配置，分别为master、slave1、slave2。配置启动信息

```c
## master
daemonize yes
pidfile /var/run/redis-16379.pid
logfile /var/log/redis/redis-16379.log
port 16379
bind 0.0.0.0
timeout 300
databases 16
dbfilename dump-16379.db
dir ./redis-workdir
masterauth 123456
requirepass 123456
    
## slave多加一个如下的配置
slaveof 127.0.0.1 16379
```

使用redis-server三个redis实例。这时会开启主从模式。

- 启动Sentinel集群

拷贝三个redis-sentinel.conf文件，并且配置节点信息。注意: 只需要配置master的信息，slave会通过INFO指令来动态获取，三份配置均相同。

```c
protected-mode no
bind 0.0.0.0
port 16380
daemonize yes
sentinel monitor master 127.0.0.1 16379 2
sentinel down-after-milliseconds master 5000
sentinel failover-timeout master 180000
sentinel parallel-syncs master 1
sentinel auth-pass master 123456
logfile /var/log/redis/sentinel-16380.log
```

使用redis-sentinel分别启动三个节点。每个Sentinel会给自身生成唯一的节点标志`fd166dc66425dc1d9e2670e1f17cb94fe05f5fc7`。

在进行主备切换后sentinel集群会自动的往redis.conf配置文件里更新对应的slaveof的信息。

# 3. Redis的集群方案

Redis的集群具有高可用、扩展性、分布式、容错等特性。一般分为Client方式和Proxy方式。

## 3.1 Client方案

客户端方式如下所示。由客户端配置各个节点的信息。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709005847.png)

- 缺点

  客户端无法动态增删服务节点，客户端维护分发逻辑。客户端无法共享连接池信息，容易造成负载不均衡。

- 优点

  不使用第三方插件，配置简单，节点之间无关联

## 3.2 PROXY模式

客户端发送请求到PROXY组件，由代理组件解析客户端的数据，并将请求转发到正确的节点，最后将结果回复到客户端。

1. 优点 

   简化了客户端的分布式逻辑，客户端接入透明。

2. 缺点

   多了一层代理层，增加了部署的复杂度和性能损耗。

一般PROXY的主流方式有2种，`Twemproxy`和`Codis`。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709132830.png)

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709132945.png)

codis是一个比较完整的redis集群的部署方式。

- codis订阅sentinel的信息来监控集群的状态
- sentinel监控redis集群的状态，并执行HA
- codis维护了集群的路由状态，可以采用一致性哈希算法或者16384哈希槽的方式来维护路由。
- codis采用主从的方式来部署，用zk来监控集群的状态并向客户端同步codis集群的信息。

## 3.3 Redis Cluster模式

客户端随机请求任意一个Redis实例，由Redis实例将请求转发给正确的redis节点。Redis cluster不是将请求转发到另一个redis节点，而在采用客户端重定向的方式。**集群的每个节点都维护了每个节点的路由信息。**

没有哨兵节点，而是集群之间通过GOSSIP协议来检测集群的状态。

- 优点

  无中心节点 数据按照槽分布在多个实例上，可以平滑的扩容，支持高可用和自动故障转移。

- 缺点

  无法支持mget操作，故障检测和转移较慢, gossip协议不如zk中心节点及时

![jiqu](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709215129.png)

### 3.3.1 集群的GOSSIP通信

集群的节点之前采用GOSSIP协议进行通信。当节点的主从发生变化后，就会彼此发送PING/PONG消息来通知全部节点的最新状态达到最终的一致性。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709225137.png)

GOSSIP的通信的流程图如下所示

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709225434.png)

在节点数很大的时候，GOSSIP协议对网络带宽的压力也会很大。 否则集群的一致性会很难保证。

### 3.3.2 集群如何故障检测

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709225607.png)

集群会检测主节点的状态，当master出现故障后会检测其slave是否具有故障转移的资格。

slave发起投票，只有master节点才有投票权。当获得超过一半的票就可以执行故障迁移操作，并广播自己的状态。





# 4. 集群负载均衡算法

分布式数据库都要解决将整个数据集 按照映射的规则到多个节点的问题，把数据集 划分到多个节点，每个节点完整的负责 整个 数据的一个子集。

负载均衡一般有如下几种方式

## 4.1 哈希取余

计算公式为hash(key) %N 计算出哈希值，在映射到一个节点上。

- 优点

  简单，一般用于数据库的分库分表策略。提前预分区将数据划分为512个表或者1024个表。后期根据负载情况将表迁移到其他数据库中。一般采用翻倍扩容，避免数据映射被打乱导致全量迁移。

- 缺点

  在节点数量发生变化时，节点的映射关系会发生变化，导致数据的迁移。(采用8个 16个这样可以避免大量的迁移)

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709221003.png)

## 4.2 一致性哈希算法

一致性哈希算法将所有的存储节点排列在一个hash环上，每个key在计算后都顺时针找到相邻的最近节点。当有节点加入或者退出时，仅仅需要迁移一部分的数据，对集群上大部分的节点不会造成影响。

- 优点 
  加入节点或者删除节点只会影响哈希环上顺时针方向的节点，对其他节点无影响。
- 缺点
  增减节点会造成哈希环上部分的数据无法命中。当只有少部分的节点时，容易造成哈希不均匀导致集群的负载不均匀。可以采用**虚拟槽**来改进。即每个节点都对应5个虚拟节点哈希到集群上。来让节点分布更加均匀。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709221505.png)

## 4.3 虚拟槽分区

**虚拟槽分区采用固定的哈希长度，将所有的数据映射到固定范围的整数集合。**Redis Cluster 的槽范围是0-16383。

**槽是集群内数据管理和数据迁移的基本单位**。

采用大范围槽的目的是为了将数据进一步做拆分和集群扩展。每个物理节点负责一定数量的槽。

如下图所示
当增加一个节点6时，会进行一次rehash的过程，将节点1-5的一部分槽分配到节点6上。如果要移除节点1，就将节点1的槽都迁移到2-5中，再讲节点1移除。

> 从一个节点将哈希槽移动到另一个节点都不会停止服务，在数据迁移的过程中集群还是可用的。但是迁移的槽可能会停止对外服务。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200709222154.png)

## 4.4 redis集群的负载均衡算法

**Redis Cluster采用的是虚拟槽分区的方式。一共是16384个槽。redis采用的是bitmap来存储该节点维护了那些槽的信息。使用16384是为了减少槽的大小，使集群在PING的时候消息体不会大。**

### 4.4.1 虚拟槽的特点

优点

- 结构了数据和节点之间的关系，简化了缩扩容的数据迁移问题。
- 每个节点都维护了集群的槽映射关系。不需要客户端和PROXY来维护槽元数据。

缺点

- 批量操作收到限制
- 对事务的支持有限
- key作为数据分区的最小单位，主从复制也只能支持一层。

# 5. redis集群的搭建

Redis集群至少要3主3从才可以达成一个高可用的状态。复制6份Redis.conf文件。并配置开启集群

```c
# redis后台运行
daemonize yes
# 绑定的主机端口
bind 127.0.0.1
# 数据存放目录
dir /usr/local/redis-cluster/data/redis-6379
# 进程文件
pidfile /var/run/redis-cluster/${自定义}.pid
# 日志文件
logfile /usr/local/redis-cluster/log/${自定义}.log
# 端口号
port 6379
# 开启集群模式，把注释#去掉
cluster-enabled yes
# 集群的配置，配置文件首次启动自动生成
cluster-config-file /usr/local/redis-cluster/conf/${自定义}.conf
# 请求超时，设置10秒
cluster-node-timeout 10000
# aof日志开启，有需要就开启，它会每次写操作都记录一条日志
appendonly yes
```

使用redis-server先将6台机器启动。

再使用`redis-trib.rb` 开始集群的初始化和哈希槽的分配。

集群创建后，`redis-trib` 会先将 `16384` 个 **哈希槽** 分配到 `3` 个 **主节点**，即 `redis-6379`，`redis-6380` 和 `redis-6381`。然后将各个 **从节点** 指向 **主节点**，进行 **数据同步**。



























参考文档

https://www.cnblogs.com/daofaziran/p/10978628.html

https://juejin.im/post/5b7d226a6fb9a01a1e01ff64