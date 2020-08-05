---
title: AQS原理
date: 2018-08-25 15:30:05
tags: 
    - java并发
    - 线程同步
    - 内存可见性
categories: java同步包
image: https://medesqure.oss-cn-hangzhou.aliyuncs.com/O84zP57OFwl3.jpg
top: 100





---

[TOC]

# 1. 简介

java中实现线程同步有synchronized和ReentrantLock方法，当然还有其他的方法。synchronized是使用`monitorenter`和`monitorexit`指令来同步代码块。而ReentrantLock是底层依赖AQS(AbstractQueuedSynchronizer)组件来实现的，即非阻塞同步队列。AQS相比于monitor可以实现多种同步方式，比如独占锁，共享锁，条件队列，公平锁等模式。在并发效率上，synchronized有自旋锁，偏向锁，轻量级锁，重量级锁的优化后，效率和ReentrantLock是差不多的。但是在同步的模式上只能是独占锁。看AQS原理前可以先思考以下几个问题

1. 线程如何访问AQS维护的资源
2. 当资源不可访问时，当前的线程如何挂起 
3. 当线程提前被中断或者其他原因退出访问资源，如何从AQS队列中退出

<!-- more -->

如图所示，AQS框架实现的一系列的同步器都是依赖于一个单独的原子变量(state)。**AQS框架的核心是维护了一个volatile的字段state和一个双向的队列，每一个请求state资源的线程都是一个Node节点，每一个节点根据得到的state的值是判断是否进行阻塞和唤醒操作**。AQS框架就是判断当state资源不能被访问就将线程加入队列中和挂起、唤醒等操作。AQS对state资源的操作是采用C  



![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200108004649.png)

从使用者的角度看，AQS是一个抽象类，只有去实现了AQS的类才能使用同步功能，AQS框架提供了基础的两个功能

1. Exclusive(独占模式，只有一个线程能执行其余线程均挂起，比如ReetrantLock)
2. Share(共享模式，多个线程可以同时执行，比如Semaphore/CountDownLatch)

针对Exclusive模式，需要实现以下的方法

1. tryAcquire(int) 尝试获取资源，成功就返回true，失败就返回false
2. tryRelease(int) 尝试释放资源，成功就返回true，失败就返回false

针对Share模式，需要实现以下的方法

1. tryAcquireShared(int) 尝试获取资源，负数表示失败；0表示成功但是没有剩余资源；整数表示成功并且剩余资源。
2. tryReleaseShared(int) 尝试释放资源，如果释放后允许唤醒后续等待节点返回true，否则返回false。

举个例子来说

1. ReentrantLock使用了AQS的独占锁模式，初始的state的值为0，表示未锁定状态。当A线程调用lock()后，会调用tryAcquire()来独占锁并将state+1。此后其他线程再次调用tryAcquire()时就会失败，直到A线程调用unlock()方法将state减少到0即释放锁为止。其他线程才有机会去获取该锁。ReentrantLock是实现了重入锁功能，在释放锁之前，A线程可以重复获取占有锁(state累加)。但是获取多少次就要释放多少次，保证state回到0。
2. CountDownLatch使用了AQS的共享锁模式。初始化有N个线程需要同步执行，那么CountDownLatch设置的state的值为N。当每个线程调用countDown()方法后，state会CAS减1。等到所有的子线程都执行完毕后(state=0)，会unpark() 唤醒主调用线程， 主调用线程会从await()函数中返回并继续后续动作。

# 2 AQS的数据结构

AQS维护了一个volatile int state(共享资源)和一个FIFO非阻塞等待队列来实现的。

## 2.1 node节点的数据结构

AQS的Node类使用waitState字段来保存节点的状态。节点的状态有以下几种

- **CANCELLED** 值为1，表示该节点处于取消状态。该节点上的线程被等待超时或者被中断，需要从同步队列中移除该Node节点。waitState的值为CANCELLED。进入该状态的节点的状态不会再发生改变。
- **SIGNAL** 值为-1，表示该节点处于唤醒状态。被标记为该状态的节点的线程释放了同步锁或者被取消，就要唤醒其后继节点。
- **CONDITION** 值为-2 和condition有关，标志该节点处于**等待队列**中，节点的线程等待在Condition中，当其他线程调用了Condition的signal()方法后，CONDITION状态的节点会转移到同步队列中，等待获取同步锁。 
- **PROPAGATE**：值为-3，与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态。
- **0状态**：值为0，代表初始化状态，head初始化条件队列时使用

在node类中有prev和next，看出AQS的同步队列是双向队列。有thread来指向当前线程，nextWaiter 如果当前的节点时共享模式，值指向一个SHARE节点。当前节点是条件队列中，值会指向下一个等待条件的节点。waitstatus表示当前节点的状态，值如下所示。

```java
static final class Node {
        // 共享锁模式
        static final Node SHARED = new Node();
        // 独占锁模式
        static final Node EXCLUSIVE = null;
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
        /**     
        * CANCELLED，值为1，表示当前的线程被取消     
        * SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark；     
        * CONDITION，值为-2，表示当前节点在等待condition，也就是在condition队列中；           
        * PROPAGATE，值为-3，表示当前场景下后续的acquireShared能够得以执行；     
        * 值为0，表示当前节点在sync队列中，等待着获取锁。     */
        // 线程的等待状态
        volatile int waitStatus;
        //前驱节点
        volatile Node prev;
        //后继节点
        volatile Node next;
        //该节点的线程
        volatile Thread thread;
        // 存储condition队列中的后继节点
        Node nextWaiter;
        //是否是共享模式
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
		
        // Used by addWaiter 
        Node(Thread thread, Node mode) {     
            this.nextWaiter = mode;
            this.thread = thread;
        }
        // Used by Condition
        Node(Thread thread, int waitStatus) { 
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

## 2.2 AQS队列结构

AQS中有3个重要的变量,head、tail、state都是volatile类型的，保证了可见性。示意图如下所示

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200629010334.png)

```java
// 队头结点    
private transient volatile Node head;     
// 队尾结点    
private transient volatile Node tail;     
// 代表共享资源    
private volatile int state;     
protected final int getState() {        
    return state;    
}     
protected final void setState(int newState) {        
    state = newState;    
}     
//CAS操作，当stateoffset的内存的值state和expect相等时将内存的值设为update
protected final boolean compareAndSetState(int expect, int update) {        
    return unsafe.compareAndSwapInt(this,stateOffset, expect, update);    
}
```

## 2.3 AQS示意图

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200629010941.png)

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20200629011053.png)

# 3 AQS源码解析

AQS的共享锁和独占锁的区别主要是独占锁是当存在资源时，FIFO队列里只有一个线程能获取资源，其他的线程都要挂起。

共享锁是当存在资源时，FIFO队列里的线程都能获得资源放行。

## 3.1 独占锁的原理

### 3.1.1 acquire()方法

注意的是tryAcquire()方法的返回值是boolean。当返回true，认为获取线程获取锁资源成功，加锁成功。

```java
public final void acquire(int arg) {
    /**
         * 整体分为四大步骤
         * 1、tryAcquire：获取锁，其实就是设置state，
         * 成功了，if语句就终止，失败了，if语句就继续下面的方法（短路）
         * 2、addWaiter：使用独占模式，
         * 将一个node节点放到AQS维护的队列的队尾
         * 3、acquireQueued：主要的阻塞方法，会进行状态的翻转和再次确认等操作，
         * 如果没有中断返回false，如果中断了，返回true，下面详细分析
         * 4、selfInterrupt：设置当前线程的中断状态
         */

    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

//tryAcquire供业务方实现，返回true或者false
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

### 3.1.2 addWaiter()方法

当调用tryAcquire()方法获取资源失败就会调用addWaiter()方法创建一个Node并加入到队列的尾部。

> 将新创建的Node节点加入到FIFO的队列尾部使用的方式是CAS(compareAndSetTail)将原先的tail改变成当前的Node。如果失败就重试

```java
private Node addWaiter(Node mode) {
    // 这里可以看出，node节点内部包含当前线程的对象
    Node node = new Node(Thread.currentThread(), mode);

    Node pred = tail;
    // 这个分支，主要想快速完成入队的操作，
    //当然，这个队列必须是不为空的且tail指针没有被修改（因为CAS在修改之后会失败）
    if (pred != null) {
        node.prev = pred;
        // 这里将队尾的指针设置成当前节点
        if (compareAndSetTail(pred, node)) {
            // 倒数第二节点的后置指针next指向当前节点
            pred.next = node;
            return node;
        }
    }
    // 一般化的入队方法
    enq(node);
    return node;
}

private Node enq(final Node node) {
    // 这里又是经典的CAS死循环，直到设置成功返回位置
    for (;;) {
        Node t = tail;
        if (t == null) {
            // 这个分支代表是全新的队列，初始化的状态
            // 设置head对象
            if (compareAndSetHead(new Node()))
                // 让尾指针等于头指针
                tail = head;
        } else {
            // 这个分支代表是全新的队列，初始化的状态
            node.prev = t;
            // 设置尾指针为当前节点
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

### 3.1.3 acquireQueued(preNode, Node)阻塞判断方法

当将节点Node加入到队列tail后就会调用acquire()方法看尝试让队列中的节点获取资源，索取失败就阻塞挂起。

acquireQueued()方法是判断队列里的节点是否能被唤醒的方法。Node节点只有能获取到state资源并且前任节点是head节点才可以放行，否则就挂起。

只有preNode节点的waitStatus状态是SIGNAL才可以挂起，如果状态是CANCEL就要将节点从队列中移除。

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    // 这又是一个死循环，会有阻塞的方法parkAndCheckInterrupt
    for (;;) {
      final Node p = node.predecessor(); //prev节点
      if (p == head && tryAcquire(arg)) {
        /**
         * 这个分支首先出现，如果当前节点的前置是头节点，那就再试试能不能获取到资源
         */
        setHead(node);// 头节点设置成当前节点
        p.next = null;  // 老的头结点要从当前队列中解开引用
        failed = false;
        return interrupted;
      }
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt()) {
        /**
         * 这个分支就是主要的阻塞分支，我们分别来看看两个方法
         */
        interrupted = true;
      }
    }
  } finally {
    if (failed)
      /**
       * 这个分支是遇到了未知的错误，或者异常，此时的failed就是true，
       * 不能正常的设置成false，那么就要将当前节点设置成取消的状态
     */
      cancelAcquire(node);
  }
}
```

### 3.1.4 shouParkAfterFailedAccquire()方法阻塞队列

当acquireQueued()方法队列中的节点获取资源失败就要调用该方法判断节点是否应该阻塞挂起。

只有前置节点的waitStatus状态为SIGNAL，当前节点才能挂起。如果是CANCEL就要移除，其他的状态要设置成SIGNAL状态。

> 如果请求队列允许被中断，那就是在被唤醒时抛出一个异常结束for循环

```java
/**
     * 这个方法主要就是读取前置节点的状态，看看是否要阻塞
     * pred：当前节点的前置节点
     * node：当前节点
     */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  // 前置节点的状态
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    // 如果前置节点已经是SIGNAL状态了，那就返回true，接下来就可以进入if的下一个方法，然后阻塞
    return true;
  if (ws > 0) { //cancel节点表示取消
    // 这个状态表示，前置节点是取消的状态，那就将指针一直往前找，找到第一个不大于0的
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
    // 注意：这种一番的设置，会将取消节点全部变成可GC的废弃链
  } else {
    // 将前置节点设置成SIGNAL，当然CAS可能是失败的，那就进入外层的死循环
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}


private final boolean parkAndCheckInterrupt() {
  /**
   * 这个方法是在shouldParkAfterFailedAcquire方法返回true之后才能执行
   * 其实就是可以阻塞方法，阻塞的条件是：前置节点是SIGNAL状态
  */
  LockSupport.park(this);// 阻塞，可中断，但是整个AQS框架不一定会响应终端状态
  return Thread.interrupted();
}

//AQS同时也实现了一个可中断的版本 区别就是在线程中断被唤醒后抛一个异常
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
//可超时的版本 就是LockSupport加了一个超时的时间，超时后线程会被中断
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 3.1.5 cancelAcquire()取消节点阻塞状态的方法

步骤如下

1. 将当前节点的状态设置为CANCEL。并将prev节点中为CACNEL状态节点都从队列中移除。
2. 如果当前节点是tail节点，就CAS将prev节点的next设置为null。将当前节点从队列中移除。
3. 如果当前接节点不是tail节点，这里需要分类讨论。
   - 如果前置节点不是head节点并且状态是SIGNAL或者设置成SIGNAL成功，那么本节点是可以获取唤醒通知的，就将prevNode的next设置为next节点将当前节点从队列中删除。(保证前置节点是SIGNAL且将当前节点从队列中移除)
   - 如果前置节点是head节点或者操作失败就唤醒当前节点的后置节点来删除当前几点。

总之，该方法的作用是将当前的节点删除并且要保证后一个几点可以得到通知。

```java
private void cancelAcquire(Node node) {
    if (node == null)
        return;
    node.thread = null;
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    Node predNext = pred.next;

    node.waitStatus = Node.CANCELLED;
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
                (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```

### 3.1.6 unparkSuccessor(Node)方法 唤醒后继节点

找到当前节点的后继第一个状态不是CANCEL的节点唤醒

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

### 3.1.7 release()方法释放资源

释放独占节点，如果释放资源成功，就将head节点的后继节点唤醒。每次只唤醒一个节点。

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

## 3.2 共享锁的原理

### 3.2.1 acquireShare()方法

提供tryAcquireShared(arg)方法供业务方实现，返回值是int。当小于0 表示没有得到足够的资源线程进入队列里。

```java
// 共享锁的入口
public final void acquireShared(int arg) {
  // tryAcquireShared同步工具类的实现方法，小于0表示没有请求到足够的资源
  if (tryAcquireShared(arg) < 0)
    doAcquireShared(arg);
}
```

### 3.2.2 doAcquireShared()进队列操作

步骤和独占锁进队列类似。具体步骤如下

1. 将当前线程包装成一个SHARE节点 并加入到FIFO队列尾部
2. 经典的for循环 查看当前节点的prev节点是不是head节点
3. 如果符合要求就去尝试获取资源，如果获取成功就将当前节点设置为head节点并将FIFO队列里的节点都尝试唤醒。
4. 如果获取资源失败，就将当前节点的线程挂起等待唤醒

```java
/**
 * 其实和独占锁的acquire大同小异
*/
private void doAcquireShared(int arg) {
  // 和独占锁类似，也是创建一个共享锁的结点，加入到队尾
  final Node node = addWaiter(Node.SHARED);
  boolean failed = true;
  try {
    boolean interrupted = false;
    // 经典的死循环
    for (;;) {
      final Node p = node.predecessor();
      if (p == head) {
        // 这个分支同样的是判断当前节点是否能够获取到足够的资源，然后正常返回运行
        int r = tryAcquireShared(arg);
        if (r >= 0) {
          // 这种是获取到足够资源的情况下，对后继节点进行一系列的逻辑处理
          setHeadAndPropagate(node, r);
          p.next = null; // help GC
          if (interrupted)
            selfInterrupt();
          failed = false;
          // 正常获取到资源的返回出口
          return;

        }
      }
      // 下面和前面的独占锁一样，不在多说
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

### 3.3.3 setHeadAndPropagate()唤醒队列的节点

共享锁当存在资源后队列里全部的节点都会被唤醒。这里必须要保证的是节点要确保能被释放并且新增的节点也被释放。采用的是for循环 步骤如下

1. 将当前Node设置为head节点，并调用doReleaseShared()唤醒头节点。
2. 每一个节点被释放完都会变成head节点，再去唤醒头结点，这样就保证了整个队列里的节点都能被释放。

```java
private void setHeadAndPropagate(Node node, int propagate) {
  Node h = head;
  // 将当前节点设置成头节点
  setHead(node);
  /**
         * 这里就是表示，如果资源足够，或者头节点为空，或者头节点为SIGNAL或者PROPAGATE状态
         * 那么要试着去释放头节点后面的结点，去试着竞争资源
         */
  if (propagate > 0 || h == null || h.waitStatus < 0 ||
      (h = head) == null || h.waitStatus < 0) {
    Node s = node.next;
    if (s == null || s.isShared())
      doReleaseShared();
  }
}

private void doReleaseShared() {
  /**
         * 我翻译了一段李老爷子的原文注释
         * 确保这种释放得以传播，即使有其他正在进行的获取或者释放资源的操作。
         * 通常情况，如果head的状态是SIGNAL，它会试图唤醒head的后继者。
         * 但是如果没有，则将状态设置为PROPAGATE，以确保在释放资源时，能够继续通知后继的节点。
         * 此外，我们必须循环，以防在执行此操作时添加新节点。
         * 另外，与唤醒后继节点的其他使用不同，
         * 我们需要知道ca是否重置状态失败，如果失败，则需要重新检查。
         */
  for (;;) {
    Node h = head;
    if (h != null && h != tail) {
      int ws = h.waitStatus;
      if (ws == Node.SIGNAL) {
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
          continue;            // loop to recheck cases
        unparkSuccessor(h);
      }
      else if (ws == 0 &&
               !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
        continue;                // loop on failed CAS
    }
    if (h == head)                   // loop if head changed
      break;
  }
}
```



# 4. AQS的应用场景

## 4.1 ReetrantLock独占锁原理

一般情况下，ReentrantLock的操作方式为

```java
reentrantLock.lock();
//do something
reentrantlock.unlock();
```

ReentrantLock保证了在同一时刻只有一个线程能获取到锁，其余的线程都要挂起等待，**直到拥有锁的线程释放了锁，被挂起的线程被唤醒重新竞争锁。**
ReentrantLock的加锁都是由AQS完成的，它只是初始化了AQS的state资源的数量和获取资源。ReentrantLock分为公平锁和非公平锁。

### 4.1.1 构造函数

获取独占锁的流程如下所示 结合ReentrantLock的源码分析 ReentrantLock的构造函数如下

```java
public ReentrantLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
}
public void lock() {
    sync.lock();
}
```

可以选择使用公平锁和非公平锁。两者的区别如下

- 公平锁 每个线程强占锁的顺序是先后调用lock方法的顺序，并依次获取锁。
- 非公平锁 每个线程强占锁的顺序不变，和调用lock方法的先后顺序无关。

公平还是非公平的区别是调用lock的时候直接获取资源还是先加入到队列的tail中。

```java
//非公平锁
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
//公平锁
final void lock() {
    acquire(1);
}
```

### 4.1.2 NonfairSync非公平锁实现原理

在Sync中最重要的就是对tryAcquire(arg)方法的override了。非公平锁获取资源的步骤如下

1. 获取state的值，如果为0 说明还没有被线程占用。就CAS将值设为1，并设置保存当前的线程。
2. 如果state的值为1，就判断是不是线程重入。是的话就将state值++。否则就认为获取资源失败，并进入队列。

```java
//非公平锁
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 4.1.3 fairSync公平锁实现原理

公平锁在获取资源的步骤如下

1. 获取state的值 如果不为0就判断是不是线程重入，走重入逻辑或者线程挂起逻辑。
2. 如果state值为0，如果队列存在，就么就要将当前的线程先加入到队列里排队获取资源。

```java
final void lock() {
    acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 4.1.4 tryRelease()方法释放资源的逻辑

ReentrantLock在释放锁的逻辑都相同。如果判断返回true，认为是释放成功了。否则释放失败

1. 判断当前的线程是不是目标线程 不是抛异常
2. 先将state资源释放release数量，如果还是大于0，那就认为是重入了。继续占有锁。
3. 如果state资源被释放为0，就认为锁被释放了。返回true

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

### 4.1.5 ReentrantLock的特性

#### 4.1.5.1 公平锁和非公平锁

这个在上面已经说明了。公平锁就是都要入队列依次排队获取资源

#### 4.1.5.2 可中断锁

走可中断的doAcquireInterruptibly()加入队列并获取资源的方法。当线程被中断时会立即抛出异常结束

```java
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}

public final void acquireInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

#### 4.1.5.3 可超时的锁

```java
public boolean tryLock(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```

## 4.2 CountDownLatch对AQS共享锁的使用

### 4.2.1 获取资源和释放资源

重写了TryAcquireShared方法，当资源值为0返回1 否则返回-1

在释放资源时当state值扣减完不为0返回true否则返回false

```java
//当state值为0时，返回1 认为获取锁成功 释放所有等待线程
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

### 4.2.1await()方法和countDown()方法

await()方法是线程获取锁资源，获取失败就被阻塞挂起。countdown()方法是释放锁资源，当state值变成0认为锁资源被释放。唤醒全部等待的线程。

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public void countDown() {
    sync.releaseShared(1);
}

```

## 5. 不同的子类对AQS的state的维护

1. ReentrantLock:  独占锁 如果state为0 说明锁是空闲的，调用tryAcquire()设置state=1，当前线程获取锁 如果state大于1，则表示当前线程获得了重入的效果，其他线程只能被park，直到这个线程进入锁的次数为0而释放原子状态 
2. Semaphare 共享锁 state记录了当前还有多少次许可证可以使用。当state小于0，则线程被阻塞。否则线程可以执行。维护了同时可以并发的线程数量
3. CountDownLatch 共享锁 闭锁是state来维护一个状态，在这个状态到达一个状态之前，所有的线程都会被park，当state=0时，则所有被park的线程都会被唤醒。
4. FutureTask 独占锁 用state来表示执行任务的线程的执行状态。当调用get()方法会获取state的值，当大于0(RUNNING)时，表示线程还在执行。，AQS就会park掉get()方法的线程。在跑任务的线程结束后会回调一个方法设置state的状态为0(FiNISHED)。然后unpark唤醒get的线程，获取执行结果。

