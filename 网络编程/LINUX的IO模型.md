title: LINUX的IO模型
date: 2018-08-14 00:38:34

tags: 
  - IO模型
  - 同步、异步、阻塞、非阻塞
  - socket编程
categories: 网络编程
image: ![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/tim-van-der-kuip-1398896-unsplash.jpg)



# 1. 简介

在网络编程中，经常会出现同步、异步、阻塞、非阻塞等概念。那么同步阻塞，同步非阻塞，异步阻塞，异步非阻塞这些不同的IO模型是如何定义的呢？本文讨论的是网络IO的情况。服务端和客户端的通信中必定会存在网络IO。在网络IO中，内核会先收集完整的TCP或者UDP数据包，再将数据从内核复制到用户的内存中。这里的IO模型是针对于服务器端的，因为客户端一般都是阻塞的。在讨论网络IO之前，需要先知道一些背景知识。

1. 用户空间和内核空间
2. 进程切换
3. 进程的阻塞
4. 文件描述符
5. 缓存IO

## 1.1 用户空间和内核空间

现在的操作系统都是采用虚拟存储器，那么对32位操作系统而言，它的寻址空间（虚拟存储空间）为4G（2的32次方）。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核（kernel），保证内核的安全，操心系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。针对linux操作系统而言，将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF），供内核使用，称为内核空间，而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF），供各个进程使用，称为用户空间。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20191213004050.png)

有了用户空间和内核空间的划分，LINUX内核结构可以分成三个部分，从底层到上层分别为硬件 -> 内核空间 -> 用户空间。**用户程序通过内核的系统调用来操作各种硬件外设**。如下图所示

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20191213004311.png)

### 1.1.1 用户态和内核态

当一个进程执行系统调用而陷入内核代码中执行时，称进行处于内核运行态（内核态）。此时处理器处于特权级别最高的（0级）内核代码中执行。**当进程处于内核态时，执行的内核代码会使用当前进程的内核栈。每个进程都有自己的内核栈**。

  当进行在执行用户自己的代码时，则称其处于用户运行态（用户态）。此时处理器在特权级最低的（3级）用户代码中运行。**当用户程序被中断程序中断时，进程也会从用户态进入到内核态。因为中断处理程序使用当前进程的内核栈**。

## 1.2 进程切换

为了能多任务执行，LINUX内核必须有能力能挂起正在CPU上运行的进程，并且恢复某个挂起进程执行。这种行为被称为进程切换。任何进程都是在操作系统内核的支持下运行，是和内核紧密相关的。

进程切换的过程如下所示

1. 保存当前进程的上下文，主要是程序计数器和其他寄存器(进程的栈)
2. 更新PCB信息
3. 把进程的PCB移入相应的内核队列，比如就绪、阻塞等队列。
4. 选择另外一个进程，更新其PCB信息。
5. 恢复该进程的上下文，继续执行。

进程的切换都是CPU密集型计算，非常的耗费CPU资源，频繁的进程切换将会导致CPU飙升等问题。

## 1.3 进程的阻塞

正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语(Block)，使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。进程进入阻塞状态时不占用CPU资源。

## 1.4 文件描述符fd

文件描述符在形式上是一个数字，表示一个索引值，指向内核为每一个进程所维护该进程打开文件的记录表。当程序或者创建一个新文件，内核向进程返回一个文件描述符。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/1576604638(1).png)

## 1.5 缓存I/O

缓存 I/O 又称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O。在 Linux 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，**读取IO时数据先从外设拷贝到操作系统的内核缓冲区中，然后才会从内核的缓冲区拷贝到应用程序的地址空间，在拷贝的过程中进程是被阻塞挂起的，直到拷贝完成**。

缓存IO的缺点是数据在传输的过程中需要在应用程序和内核之间进行多次的数据拷贝操作，这些数据拷贝带来的CPU和内存开销非常大。

# 2. IO模式

**对于一个网络IO的情况，它会涉及到两个系统对象，一个是调用这个IO的进程或者线程，另一个就是系统内核(kernel)**。当一个read操作发生时，它会经历两个阶段：

1. 等待数据准备(waiting for the data to be ready)
2. 将数据从内核拷贝到进程中(copy the date from kernel to the process)

记住这两点很重要，因为这些IO Model的区别就是在两个阶段上各有不同的情况。

## 2.1 阻塞IO模型(block I/O)

在默认的情况下，linux的所有socket都是blocking的，一个典型的读IO流程如下

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20191217004541.png)



当用户进程调用了recvfrom这个系统调用，kernel就开始了IO的第一个阶段：准备数据（对于网络IO来说，很多时候数据在一开始还没有到达。比如，还没有收到一个完整的UDP包。这个时候kernel就要等待足够的数据到来）。这个过程需要等待，也就是说数据被拷贝到操作系统内核的缓冲区中是需要一个过程的。而在用户进程这边，整个进程会被阻塞（当然，是进程自己选择的阻塞）。当kernel一直等到数据准备好了，它就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block的状态继续运行。所以，**block I/O的特点在IO执行的两个阶段都被block了**。

## 2.2 非阻塞I/O (nonblocking I/O)

在linux下，可以将socket设置为non-blocking。 对一个non-blocking socket执行读操作时的流程如下所示

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/1576515483410.png)

当用户进程发出一个recvfrom系统调用时，**如果kernel中的数据还没有准备好，那么不会阻塞进程，而是立即返回一个error**。从用户进程的角度来看，它发出一个read操作后并不需要立即等待就马上得到一个结果。用户进程判断是一个error时得到数据还没有准备好，那么它可以等待一段时间后再次发送read操作。一旦kernel中的数据准备好并且再次收到用户进程的system call，那么就执行将数据从内核空间拷贝到用户空间，然后返回。**non-blokcing IO的特点是需要不断询问kernel数据是否准备好**。

### 2.2.1 非阻塞接口和阻塞接口的区别

非阻塞接口是系统调用后立即返回，不同的返回值代表了不同的含义。阻塞接口是系统调用后必须等待数据接收完毕。

非阻塞接口不同的返回值代表的含义如下

1. recv()返回值大于0，表示接收数据完毕，返回值是接收到的字节数。
2. recv()返回0，表示连接断开
3. recv()返回-1，且errno等于EAGAIN，表示recv操作还没执行完成
4. recv()返回-1，且errno不等于EAGAIN，表示recv遇到系统错误。

服务器线程通过循环调用recv()接口，可以在单个线程内实现对所有连接的接收工作，但是循环调用recv()将大幅度推高CPU的占用率，**在这个方案中recv()的作用是检测“操作是否完成”的作用，操作系统有更为高效的方式检测“操作是否完成”**，比如select()多路复用模式，可以一次检测多个连接是否有效。

## 2.3 多路复用IO模型(IO multiplexing)

在非阻塞IO模型中，需要用户线程一直轮询recv()查看是否有数据准备好的连接，线程一直是活跃的，会浪费大量的CPU时间。这时可以采用IO复用模型，即select/epoll方式，或者称为事件驱动IO模型。

在select/epoll中，一个进程可以处理多个网络连接IO，基本原理是操作系统的select、poll或者epoll函数会不断轮询所负责的所有socket，当某个socket有数据到达时，通知用户进程，流程如下所示

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20191217011134.png)

当用户进程调用select函数时就会发生一次系统调用进入内核态,进程就被block了，同时kernel会“监视”所有该select负责的socket，任何一个socket数据准备好了，select就会返回，这时用户进程再一次read操作，将数据从内核空间拷贝到用户空间。

**所以，多路I/O复用模型的特点是能操作系统能用一个进程同时等待多个文件描述符，当这些文件描述符任意一个进入就绪状态，select函数就可以返回，而BIO和non-blocking IO都是一个进程等待一个fd**。

这个图和BIO模型没有太多的区别，并且需要使用2个系统调用(select和recvfrom)，而BIO只需要一个系统调用。但是，用select的优势是可以处理多个connection。

在连接数不是很多的情况下，使用多路I/O复用模型并不会比多线程模式的block IO模型的效率更高，可能延迟会更加严重。使用select/epoll模型的优势是能处理更多的连接，而不能使单个连接效率更高。

**在实际的多路I/O复用模型中，每一个的socket都是设置为non-blocking。然后将该IO注册到select函数当中。那么调用select的进程是阻塞的，但是处理socket的内容的进程是没有被阻塞的**。

### 2.3.1 select函数执行流程

**在网络编程中，创建一个socket设置为non-blocking得到一个fd，然后将该fd注册到select函数当中。调用slect函数就会遍历所有的fd是否就绪。就绪后可以读取和处理socket的内容**。

select函数的如下所示

```java
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

可以看出select是监视文件描述符集合，其中监视的类型有readfds、writefds、exceptfds。**调用select函数进程就会阻塞，直到有fd就绪(读就绪、写就绪、except)或者超时函数才会返回。当select函数返回后，通过遍历fdset就可以找到就绪的描述符**，流程图如下所示

1. 用户进程A注册好fd集合，调用select函数进入系统调用
2. 进程A从用户态切换到内核态，将fd集合从用户空间拷贝到内核空间。
3. 内核遍历fd集合，将进程A本身分别将其注册对应的外设等待队列上，进程A进入休眠。
4. 设备就绪唤醒队列的进程A并且返回就绪的fd，或者没有fd就绪timeout超时进程A被唤醒。
5. 进程A拷贝就绪的fd到用户空间，进程从内核态切换到用户态，select函数返回。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20191218015112.png)

select的优点是实现简单，并且大部分的平台都能移植。存在的缺点是单一进程能够监视的文件描述符的数量存在限制，在LINUX一般是1024。而且监视的数量过大后会导致select的效率降低。**因为要获取就绪的fd需要多次的select调用，存在多次的fd拷贝，需要耗费大量的性能。**

### 2.3.2 epoll函数的执行流程

epoll相比于select来说更加的灵活。epoll采用一个文件描述符来管理多个文件描述符，将用户关系的文件描述符的事件存在在内核的时间表中，这样在用户空间和内核空间只需要一次copy。

epoll函数的操作过程需要如下的三个接口

```java
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

1. epoll_create 创建一个epoll的句柄来表示监视fd集合。
2. epoll_ctl 是对将指定fd添加到监视句柄中或者从句柄中删除等操作。
3. epoll_wait 是句柄epoll返回的就绪的fd集合。

epoll的工作流程如下所示

1. 调用epoll函数，用户进程从用户态进入内核态，将fd集合从用户空间copy到内核空间。
2. 进程调用epoll_create创建一个句柄fd，再调用epoll_ctl将fd集合加入到句柄fd的监视集合中。
3. 进程遍历fd，将fd加入到事件的等待队列中，进程休眠
4. 进程超时被唤醒或者fd就绪被唤醒，查看就绪fd列表。如果没有fd就继续睡眠，否则返回就绪的fd。
5. 进程从内核态切换到用户态，返回就绪fd，epoll函数返回。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20191218020249.png)

### 2.3.3 epoll函数的优点 

在select或者poll函数中，进程只有在**调用select方法后，内核才会对监视的文件描述符进行扫描**。而epoll事件是先通过epoll_ctl来注册一个文件描述符，当文件描述符就绪时会采用**callback的回调机制**来通知进程。当进程调用epoll_wait时进程是直接挂起，等待回调，而select是需要主动去轮训所有的fd是否就绪。

epoll的优点如下

1. 监视的fd数量不会受到限制。
2. IO的效率不会随着监视的fd的数量的增长而下降。**epoll不同于select和poll的轮询的方式**，而是使用每一个fd的回调函数俩实现，**只有就绪的fd才会执行回调函数**。

如果没有大量的dead-connecttion，epoll的效率不会比select高很多，但是出现了大量的无效连接后，epoll的效率会大大提高。

## 2.4 异步IO(asynchronous IO)

linux下的asynchronous IO其实用的不多，在linux的2.6版本才引入。

用户进程发起read操作后，立即可以开始做其他的事情。从kernel的角度看，受到一个asynchronous read后，首先立即返回，不会对用户进程产生任何block。然后kernel会等待数据准备完成，然后将数据从内核拷贝到用户内存。最后，kernel给用户进程发送一个signal，告诉它read操作完成了，异步IO是真正的非阻塞的。

![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20191219010709.png)

# 3. 总结

## 3.1 IO阻塞，非阻塞，同步和异步概念

1. 阻塞和非阻塞  
   调用阻塞IO会一直block住对应的进程直到操作完成，非阻塞IO在kernel还在准备数据的情况下会立即返回。

2. 同步和异步
   POSIX中对synchronous IO和asynchronous IO的定义如下

- a synchronous IO operation causes the requesting process to be blocked until IO operation completes;
- an asynchronous IO operation does not cause the requesting process to be blocked;
  其中，operation IO指的是真实的IO操作，所以阻塞IO，非阻塞IO，IO复用都是属于同步IO，就是recvfrom这个系统调用。
  同步IO在kernel数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存，这时候进程是被block了，而异步IO在进程发起IO操作后，直接返回直到kernel发出signal告诉进程IO完成。

各个IO的模型如下所示

![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15522717138750.jpg)

从上面的图片看出，non-blocking IO和异步IO的区别非常明显，**non-blocking IO在等待数据的check阶段都不会被block，而一旦数据被准备好了还需要再次发起recvfrom操作来拷贝数据到用户空间，这时候进程是被阻塞的**。而**异步IO是全程都是非阻塞的**，在数据拷贝完成后发出通知，在此期间，用户进程无需检查IO的状态也不用拷贝数据。





















