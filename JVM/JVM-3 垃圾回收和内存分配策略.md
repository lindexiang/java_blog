---
title: 深入理解JVM虚拟机 第3章 垃圾回收器和内存分配策略  
date: 2018-08-12 17:33:18
tags: 
  - JVM原理 
  - 垃圾回收
  - 内存分配
categories: JVM虚拟机原理
image: https://medesqure.oss-cn-hangzhou.aliyuncs.com/3qr97J5afxqD.jpg
---

# 垃圾回收和内存分配策略
垃圾收集(Garbage Collection, GC)主要关注的是java堆。因为虚拟机栈，本地方法栈，程序计数器都是随着线程的产生和消失。所以这部分的内存分配是在编译器可知。对它的内存分配是确定的。对于java堆的对象实例，只有在运行期间才知道创建那些对象实例，则这部分的内存分配和回收是动态的。<!-- more -->
栈上的内存数据在线程运行结束后就会被回收。但是堆上的对象是动态分配的，完全不知道什么时候应该回收它。所以需要垃圾回收的机制。
## 引用计数法
给对象添加引用计数器，有引用就将加1，失效就减1.计数器为0就不可能再被使用。效率高，但是不能解决对象相互引用的问题。对象相互引用的问题如下所示，对象a1和a2均不能被正确回收。

```java 
    public class A{
        public Object instance = null;
    }
    A a1 = new A();
    A a2 = new A();
    a1.instance = a2;
    a2.instance = a1;
    a1 = null;
    a2 = null;
    System.gc();
```

## 可达性分析算法
通过一系列的称为"GC Roots"的对象作为起始点。当对象到GC Roots之间有可达的引用链则认为对象可用。否则就认为对象不可达，可以回收。当一个对象到GC Roots没有任何链相连就认为对象是不可用的。
如下所示，Object 5、Object 6、Object 7虽然相互关联，但是和Roots之间不可达，所以被认为是可以回收的对象。
![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20190622165552.png)

可以作为GC Roots对象包括如下几种

1. 虚拟机栈(栈帧的本地变量表)中引用的对象。
2. 本地方法栈中的JNI(native 方法)引用的对象。
3. 方法区中的常量或者类静态属性引用的对象。 // static的属性引用的对象 因为static是类加载就生成对象了

### 可达性分析

```java
1. Object aobj = new Object ( ) ;
2. Object bobj = new Object ( ) ;
3. Object cobj = new Object ( ) ;
4. aobj = bobj;
5. aobj = cobj;
6. cobj = null;
7. aobj = null;
```

第4行和第7行都会导致有对象被回收。因为aobj、bobj、cobj都是虚拟机栈的局部变量表中的reference所指的对象。而其中new的3个object是存放在java堆中的。那么如果aobj=bobj后，那么aobj所指向的object就没有gc roots的引用链可以引用，就会被回收。而cobj也会被回收。

```java
1. String str = new String("hello");
2. SoftReference<String> sr = new SoftReference<String>(new String("java"));//软引用
3. WeakReference<String> wr = new WeakReference<String>(new String("world")); //弱引用
```

第二个在内存不足会被回收，第三个一定会被回收。
总结下常用的会被回收的情况

* 显示的将引用赋值为null或者将已经指向某个对象的引用指向新的对象。
* 局部引用指向的对象

```java
    void fun() {
    .....
        for(int i=0;i<10;i++) {
            Object obj = new Object();
            System.out.println(obj.getClass());
        }
    }
```
循环每执行一次，生成的Object对象都会被回收。

* 只有弱引用与其关联的对象

```java
WeakReference<String> wr = new WeakReference<String>(new String("world"));
```

## 引用类型

1. 强引用 new出来的对象，只要引用存在，该对象就不会被回收 
2. 软引用 SoftReference 引用一些有用但并非必须的对象，当系统出现内存溢出异常之前会对软引用的对象进行回收，回收后还没有足够的内存才会内存溢出
3. 弱引用 被弱引用关联的对象只能生存到下一次的垃圾回收之前。即对象被引用了还是会被回收。WeakReferemce
4. 虚引用 无法通过虚引用来获得对象的实例，设置虚引用的目的是对象被垃圾回收时收到一个系统通知。PhantomReference

## 回收方法区
回收方法区方法区(采用永久代的垃圾回收机制)主要回收两个部分：废弃常量和无用的类。回收废弃常量比较简单。只要没有对象引用这个常量就可以将该常量移出常量池。**在常量池中有“abc”这个字符串，只要没有任何的String对象引用这个对象，该对象就会被回收。**
只有在以下情况下，类对象才能被回收:

1. 对象的实例被全部回收。
2. 加载该类的classloader被回收。
3. class对象没有被引用，无法通过反射调用该类的方法。

## 垃圾回收算法
### 标记清除算法
标记出所有需要被回收的对象，清除就是回收所有的标记对象的占用的内存。
![5EA3262E-6138-41AB-B06E-C7C9A21F4863](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/5EA3262E-6138-41AB-B06E-C7C9A21F4863.jpg)
图中看出容易产生碎片。后续无法为大的对象分配内存。
### 复制算法
将内存分为相等的2块，每次垃圾回收将存活的对象全复制到另外一般的干净的内存中，然后对整半个内存清理。 不需要考虑内存碎片等情况。缺点：内存缩小到原来的一半。
![37B31B22-A277-464D-B561-CA1B188A2808](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/37B31B22-A277-464D-B561-CA1B188A2808.jpg)
现在的虚拟机都是采用这个方法来收集新生代：新生成的对象98%是要死的。那么存活的对象很少。可以把**内存分为1个较大的Edge区域和2个Survicor空间**。比例为8:1。那么每次回收就把Survivior的对象和Eden对象的存活对象复制到另一个Survivor区间。然后清理刚刚所有的Edge和Survivor区间的对象。当一个survivor中不能存放所有的存活对象时候，**这些对象会直接通过担保机制进入到老年代**。 **当一个新生对象经过15次的垃圾回收后还存活，就将对象移动到老年代。**
### 标记-整理算法
标记复制算法在对象存活率较高时候要进行过多的复制操作，效率比较低。不想浪费50%的空间，还需要额外的空间担保分配以应对内存中100%的对象都存活的极端条件。所以在老年代中不使用这个方法，而是用标记整理算法。因为老年代的对象存活率较高
![4911FB05-3FCD-456B-A9DE-1B871B2B1BE5](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/4911FB05-3FCD-456B-A9DE-1B871B2B1BE5.jpg)
将对象标记，完成标记后将所有的存活对象都往一端移动，然后清理边界以外的内存。优点是可以充分利用内存。
### 分代回收算法
虚拟机采用分代收集算法，根据对象存活周期不同将内存划分为几个区域。将堆区域分为新生代和老年代。对新生代采用复制算法。对老年代就采用标记-清理或者标记-整理。
## 虚拟机垃圾回收算法实现
### 枚举根节点
可作为GC Roots的节点主要是全局性引用

* 常量 常量存在运行时常量池中
* 类的静态变量 静态变量存储在方法区中
* 本地变量表

有些方法区就有数百兆。那么如何快速定位常量、静态变量或者本地变量表，需要使用OopMap数据结构。
**OopMap在类加载完成后，虚拟机就把对象什么位置是什么类型的数据计算出来存储在OopMap中。在JIT编译过程中，在特定位置会记录栈和寄存器那些位置是引用**。这样GC扫描直接可以遍历Oopmap得到所有的GC Roots。

### 安全点
导致OopMap变化的指令很多。那么就要就要设定在特定的地方记录OopMap。“让程序长时间执行的特征”设立安全点。比如方法调用、异常跳转、循环跳转等。产生safepoint。
线程安全： 在gc调用时，要暂停全部线程(除了虚拟机调用的线程)。那么有两种方案。

1. 抢先式中断 如果发生gc，就把所有的线程都中断。
2. 主动式中断 如果gc需要中断，就设定标志，各线程去轮训这个标志.

java虚拟机中采用的是主动式中断，即当GC需要中断线程时，设置一个标志，各个线程执行时主动去轮训这个标志，发现中断标志为真时就自己中断挂起。**轮训标志的地方是在安全点和线程创建对象需要分配内存的地方。**
### 安全区域
使用安全点对休眠和阻塞的线程不起作用。因为阻塞的线程无法响应JVM的中断请求，JVM也无法等到阻塞的线程重新被分配CPU时间。所以对这部分阻塞的线程标记成进入安全区域。在GC时无需等待这部分线程中断。在该部分线程重新获得CPU时间后，线程需要检查等待GC回收过程完成后才能继续执行。
## 典型的垃圾回收器
### serial/serial old串行收集器
单线程收集器进行垃圾收集时，必须暂停所有用户线程。Serial收集器是针对新生代的收集器，采用的是Copying算法，Serial Old收集器是针对老年代的收集器，采用的是Mark-Compact算法。它的优点是实现简单高效，但是缺点是会给用户带来停顿。**在GC过程中需要暂停所有的用户线程直到GC结束。**
**它的优点是简单高效，没有线程切换的开销。缺点是单线程无法利用多核CPU优势。**
![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20190622174901.png)
#### ParNew收集器
ParNew收集器是serial收集器的多线程版本。在暂停用户线程后采用GC多线程去收集垃圾。**但是还是需要暂停用户线程直到GC结束。**
![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20190622175001.png)

> 并行和并发的区别
> * 并行(Parallel)是指多条垃圾回收线程并行工作，即利用多核CPU同时进行工作，不需要线程切换。此时用户线程继续处于等待状态
> * 并发(Concuurent) 用户线程和垃圾回收线程同时执行，不一定是并行，可能存在线程切换。用户线程则继续工作。

### Parallel 并行收集器
并行收集器是多个GC线程同时并行执行，充分利用CPU的多核优势。并行收集器主要是注重吞吐量，在GC工作期间用户线程需要暂停，导致系统停顿时间增大。
![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20190622180536.png)

### CMS并发收集器
CMS并发收集器是以获取最短回收停顿时间为目标的收集器。目前，很多的java应用是B/S系统的服务端，服务注重服务器的响应速度，希望系统的停顿时间最短，以给用户带来较好的体验。
CMS采用的是标记-清除算法，会产生内存碎片。

并发收集器，垃圾回收线程和用户线程可以同时进行工作。一种以获取最短回收停顿时间为目标的收集器，它是一种并发收集器，采用的是标记-清除算法。
![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20190622181050.png)

分为以下4个步骤

1. 初始标记 //暂停所有线程. 
2. 并发标记  
3. 重新标记//暂停所有线程 
4. 并发清除  
产生大量的内存碎片，需要内存紧缩操作，这个过程不能并行。在并发清除时候要和用户线程一起操作，会降低效率。
### G1收集器
G1收集器具有以下优点

1. 并发与并行
充分利用多CPU和多核的优势，使用多个CPU**并行**标记来缩短stop-world的时间。在GC收集时不停顿java工作线程，通过**并发**的方式来让程序继续运行。
2. 分代收集
采用不同的垃圾回收方法来处来达到更好的GC效果。对新生代支持标记复制算法，对老年代支持标记整理算法。
3. 空间整合
相比于CMS的标记-清理方法产生大量的空间碎片，G1回收期采用标记-整理或者标记-复制使不会产生大量的空间碎片。
4. 可预测的停顿
G1采用可预测的停顿时间。能预测垃圾回收需要的时间并控制垃圾回收的时间。G1避免了在全部的java堆回收垃圾，而是对堆进行划分，每次只收集回收价值最大的区域。即维护一个可预测时间大小的region域，根据时间要求去清除最优价值的区域。
## 内存分配和担保策略
java的内存分配就是如何在堆上分配对象，对象主要分配在新生代的Eden区域，如果启动了本地内存分配缓冲，则将按线程先在线程TLAB分配，在少数情况下也会分配在老年代。

> minor gc指新生代的垃圾回收动作，full gc指老年代的垃圾回收动作，同时会伴随一次minor gc。full gc的速度会比minor gc慢10倍以上。

1. 对象优先在Eden分配
小对象在Eden区分配，**当Eden区没有足够的空间区域分配就会发起一次Minor GC。**将eden区域和一个survivor复制到另外一个survivor，如果survivor无法容纳，就通过担保机制将对象转移到老年代。然后清空全部的空间。
2. 大对象直接进入老年代
大对象指需要连续内存空间的JAVA对象，一般是大String或者大数组，比如一个new Integer[1024]的对象就会分配在老年代。
3. 长期存活的对象会直接进入老年代
虚拟机为每个对象分配一个age计数器，对象在Eden区出生并且经过一次minor gc后存活就被移动到survivor区域，并且age+1，每次经过minor gc就会年龄+1，当age增加到一定值(默认15)，会被移动到老年代。
4. 动态对象年龄判断
在survivor区域的相同年龄的所有对象的内存总和大于survivor空间的一半，年龄大于或者等于这个对象的直接进入老年代，无需默认的15的要求。
5. 空间担保策略
发生一次minor gc后不知道有多少对象能够存活下来。在最坏的情况是所有的对象都存活。那么需要老年代的空间进行分配担保，把survivor无法容纳的对象直接进入老年代。

发生minor gc的过程是

1. 虚拟机都会先检查老年代的最大连续可用空间是否大于新生代的所有对象空间。如果大于，则Minor GC是安全的。直接gc即可。
2. 如果老年代的最大可用的连续空间小于新生代的所有对象空间，根据jvm设置是否允许担保失败，如果允许先进行一次minor gc，当存活对象过多无法容纳就会再次发起full gc。如果不允许担保失败，那么就直接进行full gc。

新生代采用复制收集算法，使用一个Survivor空间来做备份，当Minor GC后还出现大量的存活对象，survivor无法存储的对象将直接进入到老年代。**前提是老年代有足够的空间容纳这个对象**。反正老年代的空间不足就会full gc(我是这么理解的)









