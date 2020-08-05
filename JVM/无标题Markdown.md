# Redis的数据结构

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

