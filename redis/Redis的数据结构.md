
[TOC]
## 1. 简单动态字符串(SDS)

**redis的每一个键值对(key-value)都是由object组成的，键总是一个字符串对象，value可以是string，list，hash，set，sorted set等等**。

比如

```java
redis>SET msg "hello world"
OK
```

那么在redis的底层会构造一个键值对，key是“msg”，value是 “hello world”。

比如

```java
redis>RPUSH frults "apple" "banana"
Integer 3
```

则会生成一个key为 “frults”，value是一个list 

### 1.1 SDS的数据结构

```c
/*  
 * 保存字符串对象的结构  
 */  
struct sdshdr {  
      
    // buf 中已占用空间的长度  
    int len;  
  
    // buf 中剩余可用空间的长度  
    int free;  
  
    // 数据空间  
    char buf[];  
};  
```

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200617002123.png)

1. len 变量，用于记录buf 中已经使用的空间长度（这里指出Redis 的长度为5）
2. free 变量，用于记录buf 中还空余的空间（初次分配空间，一般没有空余，在对字符串修改的时候，会有剩余空间出现）
3. buf 字符数组，用于记录我们的字符串（记录Redis）

### 1.2 优点

1. 获取字符串长度的时间复杂度为O(1)
2. 杜绝缓冲区的益出，因为在修改字符串时会先检查空间是否足够
3. 使用free值记录未使用的空间，在修改字符串时减少内存重新分配次数

## 2. 链表

每一个链表节点是用listNode来表示。

```c
typedef struct listNode{
      struct listNode *prev;
      struct listNode * next;
      void * value;  
}
```

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200617002635.png)

redis的列表实现是一个双端链表，数据结构如下

```c
typedef struct list{
    //表头节点
    listNode  * head;
    //表尾节点
    listNode  * tail;
    //链表长度
    unsigned long len;
    //节点值复制函数
    void *(*dup) (void *ptr);
    //节点值释放函数
    void (*free) (void *ptr);
    //节点值对比函数
    int (*match)(void *ptr, void *key);
}
```

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200617003026.png)

**注意点: redis的链表的value是一个指针，所以链表的值可以是任意类型的对象**

## 3. 字典

redis的dict是一种用于保存键值对的抽象数据结构，一个键（key）可以和一个值（value）进行关联。比如

```java
redis>SET msg "hello world"
OK
```

创建了一个键值对("msg", "hello world")在redis中是存储在字典当中。

### 3.1 哈希表

```java
typedef struct dictht {
   //哈希表数组
   dictEntry **table;
   //哈希表大小
   unsigned long size;
   //哈希表大小掩码，用于计算索引值
   unsigned long sizemask;
   //该哈希表已有节点的数量
   unsigned long used;
}
```

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200617010801.png)

哈希表中是一个dictEntry的table。和HashMap是一样的方式。并且使用数组+链表的形式来保存冲突的键值对。

### 3.2 哈希表节点

dictEntry的节点定义如下

```c
typeof struct dictEntry{
   //键
   void *key;
   //值
   union{
      void *val;
      uint64_tu64;
      int64_ts64;
   }   struct dictEntry *next;
}
```

- key保存的是键值对的键
- v保存着键值对的值，可以是一个指针，int类型的整数
- next指针是用来解决键冲突的问题。

### 3.3 dict字典的数据结构

```c
typedef struct dict {    // 类型特定函数
    dictType *type;    // 私有数据
    void *privedata;    // 哈希表
    dictht  ht[2];
    // rehash 索引
    in trehashidx;
}
```

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200617012225.png)



ht 属性是一个包含两个项（两个哈希表）的数组。

**当哈希表的操作后保存的键值对会变化，为了让哈希表的负载因子维持在一个合理的范围之内，需要对哈希表的大小进行相应的扩展和压缩。可以通过rehash操作完成**。

负载因子的计算公式=哈希表保存的节点数量/哈希表的大小。

**当负载因子过大，说明链表冲突严重。当负载因子过小，说明空间浪费了**。

#### 3.3.1 redis的渐进式rehash

在进行拓展或者压缩的时候，可以直接将所有的键值对rehash 到ht[1]中，这是因为数据量比较小。在实际开发过程中，这个rehash 操作并不是一次性、集中式完成的，而是分多次、渐进式地完成的。

渐进式rehash 的详细步骤：

1. 为ht[1] 分配空间，让字典同时持有ht[0]和ht[1]两个哈希表
2. 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash 开始
3. 在rehash 进行期间，每次对字典执行CRUD操作时，程序除了执行指定的操作以外，还会将ht[0]中的数据rehash 到ht[1]表中，并且将rehashidx加一
4. 当ht[0]中所有数据转移到ht[1]中时，将rehashidx 设置成-1，表示rehash 结束 

**采用渐进式rehash 的好处在于它采取分而治之的方式，避免了集中式rehash 带来的庞大计算量**。

## 4. 跳跃表

- 跳跃表是基于单链表加索引的方式实现
- 空间换时间的方式提升查询速度
- Redis的有序集合在节点元素较大或者元素较多时使用跳跃表实现
- Redis的跳跃表实现是通过zSkipList和zSkipListNode组成，前者保存跳跃表的信息，包括头结点、尾节点、长度等。后者是表示一个跳跃表节点，由score、object、索引数组等信息。
- Redis的跳跃表的层高是在1-32之间的随机数
- 在同一个跳跃表中，多个节点的分数可以相同。但是分数是有顺序的。

跳跃表是一种有序的数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

对于一个链表来说，即使存储的数据是有序的，如果需要在其中查找某个元素，也只能从头到尾遍历，时间复杂度在O(n)。因此，我们想要高效的查找某个元素，需要考虑创建索引的方式，每一个节点都网上创建索引来支持快速定位查找。

如下图所示，我们搜索55只需要一次查找即可。搜索46需要需要6次查找。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200620210531.png)



### 4.1 跳跃表的性能

如果有n个元素，因此是2分，所以层数应该是logn层。对于查找一个元素，我们需要遍历所有的层2遍，所以时间是2logn。所以跳跃表的时间复杂度是O(logn)

我们并不需要通过复杂的操作调整连接来维护这样完美的跳跃表。有一种基于概率统计的插入算法，也能得到时间复杂度为O(logn)的查询效率，这种跳跃表才是我们真正要实现的。

### 4.2 跳跃表的插入

在插入一个节点时，首先需要查找到该节点，时间复杂度为O(logn)。然后使用随机数生成法来获取新元素插入的最高层数。先估计n的数量，然后定义跳跃表的最大层数为logn。

定义最大层数为maxLevel，那么底层插入的概率为1，最高层的插入概率为1/2^maxLevel。因为每次增加一个索引都要抛一次硬币。所以概率为1/2^n

比如maxLevel=4，则随机数的范围为0-15。则r小于8 的概率为1/2，小于4的概率为1/4,小于2的概率为1/8 小于1的概率为1/16。底层是一定要插入的。

### 4.3 redis中的跳跃表实现

Redis使用跳跃表作为有序集合键的底层实现之一,如果一个有序集合包含的**元素数量比较多**,又或者有序集合中元素的**成员是比较长的字符串**时, Redis就会使用跳跃表来作为有序集合健的底层实现。

跳跃表在链表的基础上增加了多级索引以提升查找的效率，但其是一个空间换时间的方案，必然会带来一个问题——**索引是占内存的。原始链表中存储的有可能是很大的对象，而索引结点只需要存储关键值值和几个指针，并不需要存储对象，因此当节点本身比较大或者元素数量比较多的时候，其优势必然会被放大，而缺点则可以忽略**。

Redis中的跳跃表由zskipListNode和skipList组成。zskiplistNode结构用于表示跳跃表节点,而 zskiplist结构则用于保存跳跃表节点的相关信息,比如节点的数量,以及指向表头节点和表尾节点的指针等等

```c
// 跳表节点结构体
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    // 节点数据
    robj *obj;
    // 分数，游戏分数？按游戏分数排序
    double score;
    // 后驱指针
    struct zskiplistNode *backward;
    // 前驱指针数组TODO
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        // 调到下一个数据项需要走多少步，这在计算rank 的非常有帮助
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    // 跳表头尾指针
    struct zskiplistNode *header, *tail;
    // 跳表的长度
    unsigned long length;
    // 跳表的高度
    int level;
} zskiplist;
```



![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200621005900.png)

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200621010336.png)

## 5. redis的压缩列表

压缩列表是redis自己设计的一种数据存储结构，类似于数组。但是通过一片连续的内存空间来存储数据。数组的每个元素大小相同。但是如果要存储不同长度的字符串，就需要用最大长度的字符串大小作为元素大小(假如20个字节)，存储小于20个字节长度的字符串就会浪费存储空间。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200621200320.png)

但是在遍历的时候不知道每个元素的大小是多少，所有无法计算出下一个节点的具体位置。所以需要给每个节点增加一个length的属性。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200621201502.png)

这样我们在遍历节点的时候就知道每个节点占用的内存了。

### 5.1 Redis的压缩列表

**zipList是redis的列表和哈希的底层实现之一。当一个列表或者哈希表只有包含少量的值或者是长度比较小的字符串，redis都会使用压缩列表作为底层的实现。**

1. redis的列表底层实现

   - 当列表对象保存的所有字符串元素的长度都小于64字节
   - 列表对象保存的元素数量小于512个 

   只有同时满足这两个条件的列表才会使用zipList实现，否则使用linkedList实现。

2. redis的哈希对象底层实现

   - 当哈希对象保存的键值对的键和值的字符串长度都小于64
   - 哈希对象保存的键值对数量小于512个

   满足这两个条件的哈希对象用zipList实现，否则使用hastable实现。

3. redis的有序列表实现

   - 当有序集合保存的元素数量小于128个
   - 有序集合保存的所有元素成员的长度小于64字节

   满足这两个条件的有序列表是使用zipList实现。否则使用skipList实现。

