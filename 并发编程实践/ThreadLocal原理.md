---
title: threadLocal原理解析
date: 2018-08-17 00:06:03
tags: 
  - ThreadLocal 
  - 并发编程
categories: 并发编程
image: ![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/t01545ea15b57dd1543.jpg)

---

# 1. 简介

> 在工作中使用到了ThreadLocal变量，但是对其原理不是非常的清楚，只是知道可以保存一个共享变量到本地线程的副本，线程之间不会竞争访问该变量。具体到在原理层面上如何去实现，还有ThreadLocal引发的内存泄漏问题都不是非常清楚。这个文章将会讲述下原理。
> <!--more-->

ThreadLocalLocal的用法如下所示：

```java
public class ThreadLocalTest {

    //创建一个ThreadLocal对象，设置初始值为3
    private ThreadLocal<Integer> tlA = ThreadLocal.withInitial(() -> 3);

    private ThreadLocal<Integer> tlB = ThreadLocal.withInitial(() -> 3);

    //信号量 每次允许一个线程进入
    Semaphore semaphore = new Semaphore(1);

    public class Worker implements Runnable {

        @Override
        public void run() {
            try {
                Thread.sleep(1000);
                semaphore.acquire();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            int valA = tlA.get();
            System.out.println(Thread.currentThread().getName() + "tlA 的初始值 = " + valA);
            valA = new Random().nextInt(100);
            tlA.set(valA);
            System.out.println(Thread.currentThread().getName() + "tlA 的新值 = " + valA);

            int valB = tlB.get();
            System.out.println(Thread.currentThread().getName() +"tlB 的初始值 = "+ valB);
            valB = new Random().nextInt(100);
            tlB.set(valB);
            System.out.println(Thread.currentThread().getName() +"tlB 的新值 =  "+ valB);
            semaphore.release();
        }
    }

    /*创建三个线程，每个线程都会对ThreadLocal对象tlA进行操作*/
    public static void main(String[] args){
        ExecutorService es = Executors.newFixedThreadPool(3);

        ThreadLocalTest tld = new ThreadLocalTest();
        es.execute(tld.new Worker());
        es.execute(tld.new Worker());
        es.execute(tld.new Worker());
        es.shutdown();
    }

}
```
运行结果如下所示：

```java
pool-1-thread-3tlA 的初始值 = 3
pool-1-thread-3tlA 的新值 = 76
pool-1-thread-3tlB 的初始值 = 3
pool-1-thread-3tlB 的新值 =  73
pool-1-thread-1tlA 的初始值 = 3
pool-1-thread-1tlA 的新值 = 47
pool-1-thread-1tlB 的初始值 = 3
pool-1-thread-1tlB 的新值 =  39
pool-1-thread-2tlA 的初始值 = 3
pool-1-thread-2tlA 的新值 = 96
pool-1-thread-2tlB 的初始值 = 3
pool-1-thread-2tlB 的新值 =  80
```
从运行结果来看，每次调用`ThreadLocal`对象的`get`方法都得到了初始值3，让3个线程按照顺序执行，从结果看`pool-1-thread-1`线程结束后设置的`tlA`的新值对`pool-1-thread-3`没有影响，线程3还是得到的是`ThreadLocal`对象的初始值3。相当于把该`ThreadLocal`对象当成是本地变量一样，但是该变量其实是一个共享全局变量。

接着对上述的代码做一些简单的改变, 将`main`函数改变线程池的容量大小为1

```java
/*创建三个线程，每个线程都会对ThreadLocal对象tlA进行操作*/
    public static void main(String[] args){
        ExecutorService es = Executors.newFixedThreadPool(1);
        ThreadlocalTest tld = new ThreadlocalTest();
        es.execute(tld.new Worker());
        es.execute(tld.new Worker());
        es.execute(tld.new Worker());
        es.shutdown();
    }
```

运行结果如下

```java
pool-1-thread-1tlA 的初始值 = 3
pool-1-thread-1tlA 的新值 = 19
pool-1-thread-1tlB 的初始值 = 3
pool-1-thread-1tlB 的新值 =  79
pool-1-thread-1tlA 的初始值 = 19
pool-1-thread-1tlA 的新值 = 86
pool-1-thread-1tlB 的初始值 = 79
pool-1-thread-1tlB 的新值 =  15
pool-1-thread-1tlA 的初始值 = 86
pool-1-thread-1tlA 的新值 = 68
pool-1-thread-1tlB 的初始值 = 15
pool-1-thread-1tlB 的新值 =  46
```
从运行结果中看出`tlA`的值被多个线程共享了，其实是因为线程池用的都是同一个线程，所以访问的是共享的变量。 接着我们看其实现原理
# 2. ThreadLocal的源码解析

## 2.1 ThreadLocalMap类

在`ThreadLocal`类中定义了一个内部类为`ThreadLocalMap`, 可以看出`threadLocalMap`的内部结构和`HashMap`的结构类似，也是维护了一个Entry的数组。**Entry的key是ThreadLocal类型，value是具体的对象**。所以相当于一个Thread中使用Map结构保存多个ThreadLocal的对象。

**WeakReference如字面意思，弱引用， 当一个对象仅仅被weak reference（弱引用）指向, 而没有任何其他strong reference（强引用）指向的时候, 如果这时GC运行, 那么这个对象就会被回收，不论当前的内存空间是否足够，这个对象都会被回收**

**Entry继承了`WeakReference`，且设置key为弱引用，`WeakReference`是在`gc`时一定会被回收的对象，，因为`ThreadLocal`是线程级的变量，当线程结束后该对象应该被回收。`softReference`是在gc后内存不足才会再gc一遍回收软引用指向的对象**。

```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);//调用weakReference的构造函数 
            value = v;
        }
    }
    
    private Entry[] table;
    
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
         table = new Entry[INITIAL_CAPACITY];
         int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
         table[i] = new Entry(firstKey, firstValue);
         size = 1;
         setThreshold(INITIAL_CAPACITY);
    }
    
    //通过key的hash值来定位在table的槽位置
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    } 
}
```

## 2.2 ThreadLocal类

首先，在Thread类中维护了一个ThreadLocalMap对象，如下所示

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

**该`ThreadLocalMap`的Map结构中key是ThreadLocal对象，value是ThreadLocal对象存的值**。

### 2.2.1 ThreadLocal的get方法

get()方法的操作步骤如下所示

1. 获取当前的线程t
2. 获取线程t的内部的ThreadLocalMap对象
3. 如果该Map对象存在，将ThreadLocal对象本身作为key去查询Map的value值
4. 如果该value存在就直接返回value，否则就设置初始值并返回。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

```

### 2.2.2 ThreadLocal的初始化value方法

初始化value的过程如下所示

1. 调用initialValue()方法生成一个初始化的object
2. 获取当前Thread的ThreadlocalMap对象。
3. 将ThreadLocal对象作为key，初始化的object为Map的value保存。

```java
//设置ThreadLocal的初始值
private T setInitialValue() {
    T value = initialValue(); //这个方法可以被重写，设置自己的初始值
    Thread t = Thread.currentThread();
    ThreadLocalMap map = t.threadLocals;
    if (map != null)
        map.set(this, value);
    else
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    return value;
}
```

### 2.2.3 ThreadLocal的set方法

set方法的操作步骤如下

1. 获取当前的线程
2. 获取线程的ThreadLocalMap对象
3. 将ThreadLocal作为key，将传入的对象作为value保存在Map当中。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

### 2.2.4 ThreadLocal的remove方法

那么remove方法的思路也是非常类似，只要remove当前的线程的ThreadLocalMap的以该ThreadLocal对象作为key的Entry即可。

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

# 3. ThreadLocal造成的内存泄漏

我们在前面说了ThreadlocalMap类中的Entry是继承了`WeakReference`，设置了Entry的key是弱引用，即ThreadLocalMap中存储的ThreadLocal对象是弱引用，而Entry的value是被强引用的。**那么在gc的时候会发生什么事情？？？ Map的key会被回收掉，而value可能还是存在的，那么就会有内存泄漏的风险**。

考虑一种情况，在一个方法内申明一个ThreadLocal对象，并设置了value值。当方法运行结束时，该threadLocal对象将没有被栈的变量指向。那么在gc时，会出现上图中所示的key为null，value存在的情况而且该value将无法被访问。**因为Thread还在运行，所以val1被ThreadLocalMap强引用而不会被回收**，那么就出现了内存泄漏的情况。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/1577210218247.png)

在get方法中的getEntry方法中存在如下的一段代码。当key为null时，会调用expungeStaleEntry()方法去遍历删除所有的key为null的Entry方法。**在调用get方法，set方法，remove方法，在key为null时会删除所有的为null的key**

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
 
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len); //其实就是i++
        e = tab[i];
    }
    return null;
}

//遍历删除所有key为null的Entry
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            //重新hash
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

上述的方法并不能保证解决内存泄漏的问题，因为在调用get方法是不一定能获得key为null的对象。当线程结束时，ThreadLocalMap对象会被回收，那么完美。但是使用线程池时，如果ThreadLocal对象被回收，而线程是回收待使用，则value会一直存在堆中无法被访问。内存就会被一直泄漏。
使用线程池时使用不当还会发生bug。**当定义一个static的ThreadLocal对象，使用线程池，在线程中set了一个ThreadLocal对象。那么下一个线程会得到上一个线程的value，造成bug。就像最开始的代码中的运行结果。所以在线程结束时，手动remove掉该ThreadLocal。** 