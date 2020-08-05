---
title: Happens-Before
date: 2018-08-25 15:30:05
tags: 
    - JVM
    - 线程同步
    - 内存可见性
categories: JVM虚拟机原理
image: http://pbhb4py13.bkt.clouddn.com/wallls.com_175256.jpg
top: 100

---

# 1. happens-before简介
在看JVM的内存模型时对内存可见性一直有个问题，线程A获得锁，更新了对象内容A，释放锁，线程B锁住，为什么能获得对象A的最新值。还有双重检查生成单例时为什么需要把instance设置成volatile类型。问题的本质是线程同步的原子性，可见性，有序性的实现原理。<!--more-->我在看了[Java 使用 happen-before 规则实现共享变量的同步操作](http://ifeve.com/java-%E4%BD%BF%E7%94%A8-happen-before-%E8%A7%84%E5%88%99%E5%AE%9E%E7%8E%B0%E5%85%B1%E4%BA%AB%E5%8F%98%E9%87%8F%E7%9A%84%E5%90%8C%E6%AD%A5%E6%93%8D%E4%BD%9C/ )。这个文章后有一种恍然大悟的感觉，终于搞明白了。

在深入理解JVM虚拟机文章中说过一句话，JMM(java内存模型)中的happens-before(hb规则)规则定义了java多线程操作的有序性和可见性，防止编译器重排序对结果的影响。 

> 官方文档描述  
一个变量被多个线程读取并且至少被一个线程写入时，如果读操作和写操作没有HB规则，那么将会产生数据竞争问题。为了保证操作B能看到操作A的结果(无论A和B是否在一个线程)，那么A和B之间必须满足HB规则，如果没有，将会导致重排序。

总结起来就是2点

- 如果满足HB规则，那么能保证可见性和有序性。比如线程加锁，同一个锁时，顺序为线程A-线程B，那么线程B一定能看到线程A的任何修改结果。缓存和主内存之间的关系会失效。
- HB规则的实现就是根据**MESI缓存一致性和内存屏障**实现的。JVM层面做了这一层的工作。

# 2. 缓存一致性和java内存模型(JMM java memory model)
在谈论java线程模型时必须要了解缓存一致性原则，这个是java内存模型的基础。java内存模型是一个概念模型，底层是寄存器、缓存内存、主内存、CPU之间的互相协作。

## 2.1 缓存一致性协议(MESI) <a id="jump"></a>
每个CPU都有属于自己的高速缓存，在需要同步的情况下，如果在缓存中更新了数据后，其他CPU读取该共享变量的缓存后就会出现**缓存不一致**的错误。这个时候就需要MESI协议来实现缓存一致性。
可以采用LOCK#信号来对总线进行锁定，一个CPU在总线上输出该信号后，其他CPU的请求将会被阻塞，则该CPU可以独自共享内存，但是该方法的开销太大，之后的计算机一般采用**缓存锁定**的方式。
MESI代表缓存数据的4种状态的名字，分别为Modified, Exclusive, Shared, Invalid

- Modified  
被修改的缓存。该缓存在本CPU中有缓存数据,其他CPU中没有，对其他缓存中的值是已经被修改过的，但是没有更新到内存中。
- Exclusive   
独占的。处于该状态的缓存，在本CPU中有缓存，并且数据没有修改，在内存中一致。
- Shared  
共享的。处于这个状态的数据在多个CPU中都有缓存，且和内存一致。
-Invalid   
失效的数据。缓存的数据已经失效，或者不在缓存中。

> 嗅探技术  
嗅探能够嗅探到其他处理器访问主内存和它们的内存缓存

缓存行在以上的4种状态的基础上，通过“嗅探”的技术完成以下功能 

- 一个处于M状态的缓存行，必须时刻监听所有试图读取该缓存行对应的主内存地址的值，如果监听到，则必须在此操作执行前将缓存行写回CPU中。
- 处于S状态的缓存行，必须监听使该缓存行无效或者独占该缓存行的请求，监听到后将该缓存行设置为I
- 处于E状态，必须监听其他试图读取该缓存行的主内存地址的操作，监听到，将该缓存行的状态设置为S
- 只有E和M状态可以进行写操作并且不需要额外操作，想要对S状态的缓存字段写操作，先发送一个RFO广播，该广播可以让其他CPU缓存中的相同的字段的状态变成I

通过以上的机制可以保证处理器的读写操作是原子性的，并且读到的数据都是最新的，即内存可见性。

> 总结以上的EMSI协议，其实就是能通过使缓存失效和读取变量强制从主内存中读取的方式来保证了内存的可见性。类似的，java中的锁和volatile也是这样的原理.

## 2.2 java并发编程中的几个原则
在使用synchorized和RenntrantLock时，我们只关注了它们的原子性，其实它们的可见性和有序性更加的重要。**可见性保证了同步的两个线程读取的变量的是最新值**。**有序性保证了线程的先后顺序**。原子性就是你们所理解的那样子。

1. 原子性  
Java主要提供了**锁机制以及CAS操作实现原子性**，对于单个读/写操作是通过LOCK#信号或“缓存锁定”实现的。除此之外，long和double类型的变量读/写是非原子性的，每次都只读/写32位数据，所以一个单个的读/写操作就变成了两个读/写操作，有可能在只读/写了其中32位操作后CPU就被其他线程抢占到。
2. 可见性  
每个线程都有私有的缓存。java中提供了volatile保证了内存的可见性，底层通过了Lock#或者缓存锁定实现。
3. 有序性  
编译器和处理器会对代码进行排序，排序包括了 1. 语句的执行顺序重排序。 2.指令集并行的重排序，多个CPU协同读取 3.内存系统的重排序 缓存和内存的数据同步存在时间差。

## 2.3 内存屏障 <a id="jump1"></a>
内存屏障是一个CPU指令，java编译器会在生成指令的适当位置插入内存屏障指令来禁止特定类型的处理器重排序，作用有2个

- 防止指令之间的重排序
- 强制把写缓冲区/高速缓存中的脏数据等写回主内存，让缓存中相应的数据失效  

### 2.3.1 硬件层的内存屏障
分为两种：Load Barrier 和 Store Barrier即读屏障和写屏障

- Load Barrier  
在指令前插入Load Barrier，可以让高速缓存中的数据失效，强制从新从主内存加载数据
- Store Barrier  
在指令后插入Store Barrier，能让写入缓存中的最新数据更新写入主内存，让其他线程可见

### 2.3.2 java内存屏障

- LoadLoad屏障  
对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕
- StoreStore屏障  
对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见
- LoadStore屏障  
对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕
- StoreLoad屏障  
对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能

### 2.3.3 final语义中的内存屏障：

- 新建对象过程中，构造体中对final域的初始化写入和这个对象赋值给其他引用变量，这两个操作不能重排序  
- 初次读包含final域的对象引用和读取这个final域，这两个操作不能重排序（先赋值引用，再调用final值）

# 3. HB规则

1. 程序次序原则: 在一个线程内，代码按照顺序执行
2. 管程锁定规则: 在同一个monitor上，unlock操作时间上先行发生于后面的lock操作
3. volatile变量规则: 对一个volatile变量的写操作先于读操作
4. 线程启动原则: Thread的start()先于该线程的任何操作
5. 线程终止原则: Thread的所有操作都先于线程的终止检测。可以通过Thread.join()和Thread.isAlive()的返回值检测线程是否已终止
6. 线程终端规则: 线程的interrupt()方法先于中断线程检测到中断事件的发生，即可以使用interrupted()方法检测到线程是否被中断了。
7. 对象终结原则: 对象构造函数执行完毕先于finilized()方法
8. 传递性: **A先于B，B先于C。可以推断出A先于C**

**只要满足如上的8条规则，都能保证后面操作的读线程能读取到前面写线程的最新值，即保证了可见性。**其中对**传递性的规则是最重要的**，如果A HB B， B HB C，那么A的操作共享变量的结果对C都是可见的。实现可见性的原理是通过缓存一致性协议(MESI)和内存屏障(Memory Barrrier)。

## 3.1 同步的实现解析<a id="jump2"></a>
Happens-Before的排序规则十分的强大，一般是使用happens-Before和监视器锁或者volatile变量的规则结合来保证对某个未被锁保护变量的访问。

如下代码是线程同步的简单列子，没有使用锁，仅仅通过volatile变量就保证了线程同步。相比于锁。没有线程的阻塞，极大的提升了效率。

```java
public class HappenBeforeTest{

    static int num = 0;

    static volatile boolean flag = false;

    class task1 implements Runnable {

        @Override
        public void run() {
            while (num < 100) {
                if (!flag && (num == 0 || ++num % 2 == 0)) {
                    System.out.println(num);
                    flag = true;
                }
            }
        }
    }

    class task2 implements Runnable {

        @Override
        public void run() {
            while (num < 100) {
                if (flag && (++num % 2 != 0)) {
                    System.out.println(num);
                    flag = false;
                }
            }
        }
    }

    public static void main(String[] args) {

        Thread thread1 = new Thread(new HappenBeforeTest().new task1());
        Thread thread2 = new Thread(new HappenBeforeTest().new task2());

        thread1.start();
        thread2.start();

    }

}
```
以上的代码会按照循序打印出0~100的数字，但是num变量并不是volatile，并且num++也不能保证原子性，会有可见性问题。**问题是为什么t1更新了num，t2能感知到**。

其实以上代码执行的顺序如下
![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20190730012944.png)
可以使用Happens-Before来完成可见性分析。

1. t1 num++，然后修改了volatile变量 则1 HB 2
2. t2 读取了修改后的volatile变量 则2 HB 3
3. t2 读取num的变量，并执行了num++ 则3 HB 4
4. 根据传递性原则，可以推断到1 HB 4

根据以上的分析，可以得到t2 对 t1的操作结果具有可见性，不用锁也能得到正确的线程执行顺序。即使num没有进行加锁访问。

3个线程按顺序打印  
3个线程按照循序打印ABC各3次还可以使用一下的方法，只是利用了volatile的可见性和HB规则。但是使用while去一直循环查询感觉效率并没有wait-notify高。

```java
public class HappenBeforeTest{

    static int num = 0;

    static volatile int flag = 0;

    class task1 implements Runnable {

        @Override
        public void run() {
            while (num < 9) {
                if (flag == 0) {
                    System.out.println("A");
                    num++;
                    flag = 1;
                }
            }
        }
    }

    class task2 implements Runnable {

        @Override
        public void run() {
            while (num < 9) {
                if (flag == 1) {
                    System.out.println("B");
                    num++;
                    flag = 2;
                }
            }
        }
    }

    class task3 implements Runnable {

        @Override
        public void run() {
            while (num < 9) {
                if (flag == 2) {
                    System.out.println("C");
                    num++;
                    flag = 0;
                }
            }
        }
    }

    public static void main(String[] args) {

        Thread thread1 = new Thread(new HappenBeforeTest().new task1());
        Thread thread2 = new Thread(new HappenBeforeTest().new task2());
        Thread thread3 = new Thread(new HappenBeforeTest().new task3());
        thread1.start();
        thread2.start();
        thread3.start();

    }

}

```

## 3.2 其他规则实现同步
以下的代码也能保证a的正确访问，因为JMM已经保证了访问的先后顺序和可见性。

```java
//Thread的join()方法保证了可见性
static int a = 1;

  public static void main(String[] args){
    Thread tb = new Thread(() -> {
      a = 2;
    });
    Thread ta = new Thread(() -> {
      try {
        tb.join();
      } catch (InterruptedException e) {
        //NO
      }
      System.out.println(a);
    });

    ta.start();
    tb.start();
  }
  
  //Thread的start()方法保证可见性
  static int a = 1;

  public static void main(String[] args){
    Thread tb = new Thread(() -> {
      System.out.println(a);
    });
    Thread ta = new Thread(() -> {
      tb.start();
      a = 2;
    });

    ta.start();
  }
```

# 4. volatile双重检查实现单例原理
上面说过了volatile的作用一共有2个。

1. 可见性
volatile变量写完会立即同步到主内存，并且使其他缓存失效，只能从主内存中读取，这个是由内存一致性协议(MESI)决定。见 [缓存一致性协议](#jump)
2. 防止指令重拍序
volatile变量会在代码编译时插入LOCK#指令，该操作相当于一个[内存屏障](#jump1)，内存屏障能保证重拍序无法将后面的指令重拍序到内存屏障之前。

##4.1 双重检查锁(DCL)
这个DCL之前在分析<a href="http://medesqure.top/2018/08/07/JVM-12/">java内存模型</a>时已经说过，还是会存在对象没有构造完全的风险。

```java
public class Singleton {
	
	private static Singleton instance = null;

	private int age;

	public static Singleton getInstance() {
		if(instance == null) { //1
			synchonorized(Singleton.class) { //2
				if(instance == null) { //3
					instance = new Singleton(); //4
				}
			}
		}
		return instance;  //5
	}

	public Singleton() {
		this.age = 18;
	}

	public int getAge() { //6
		return age;
	}
}
```

对于步骤4，看上去只有`instance = new Singleton()`一个操作，但是其实至少有3个步骤

1. 在堆中开辟一块新的内存(new)
2. 调用对象的构造函数对内存进行初始化(invokespecial)
3. 将内存的引用赋值给变量(astore)

但是存在指令重拍序时，很可能发生了如下的执行顺序，1-3-2。这个没有疑问。
重点来了，我们来好好分析一下可能出现的对象构造不完全的情况。

1. 线程A执行到4，执行new的顺序恰好是1-3-2 
2. 当其执行完3时，这个时候该线程的CPU时间被剥夺分配给线程B，而B恰好执行了步骤1.
3. 因为1不是同步操作，所以线程B判断i
f(instance == null)返回false，因为这个时候这个引用已经被分配值了。所以直接返回了一个instance。

不管B是不是会去访问age变量，但是一个没有被构造完全的对象被引用了，就存在了风险。

但是使用了volatile之后，`instance = new Singleton()`的执行顺序一定会是1-2-3，从Happens-Before的规则出发，instance对象的初始化一定会被线程B观察到，所以才会是正确的构造结果。**不会导致未被完全构造完成的对象发布出去**。

# 5. 总结
这个文章写得有点多了，最后总结一下吧。使用HB规则能简单的推导出上一个操作对下一个操作的可见性。
这个特性在使用ReentrantLock也被使用到了。所以其也可以实现线程同步的原子性，可见性，有序性。其在内部使用了volatile的state状态来定义状态，每次操作共享变量时会先读取state变量，这个就和[同步的实现解析](#jump2)一样了。因为CAS只是保证了赋值的原子性。
其实并发容器中大部分都是用了这个HB来保证可见性，CountDownlatch,Semaphore，Future等等。

最后，啰嗦一句，使用HB规则去判断可见性，是java内存模型的精髓。


