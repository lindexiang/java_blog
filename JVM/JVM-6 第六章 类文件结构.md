---
title: 深入理解JVM虚拟机 第6章 类文件结构
date: 2018-07-27 09:32:51
tags: 
  - JVM原理 
  - 类文件结构
categories: JVM虚拟机原理
image: https://medesqure.oss-cn-hangzhou.aliyuncs.com/undertale-frisk-flowers-anime-style-back-view-29102.jpg
---

# 深入理解Java虚拟机 第六章 类文件结构
jvm是平台无关性。这些虚拟机都可以载入和执行同一种平台无关的字节码，从而实现了程序的“一次编写，到处运行”。各个平台的虚拟机都使用统一的程序存储格式--字节码(ByteCode)。
![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20190623010732.png)
JVM虚拟机只和Class文件这种二进制文件格式所关联。Class文件中包含了JVm需要的指令集合符号表。所以只要任何语言使用编译器将其语言编译成存储字节码的class文件就可以在虚拟机中运行。
<!--more-->

## class类文件的结构

1. class文件是以8字节为基础的二进制流，各个数据都是按一定的顺序排列，如果需要占用8字节以上的空间数据，按照高位在前分割存储
2. class文件的存储结构只有2种，无符号数和表

整个class文件本质上就是一张如下的表
![1](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-19%20%E4%B8%8B%E5%8D%882.46.56.png)



```java
ClassFile {
              u4             magic;
              u2             minor_version;
              u2             major_version;
              u2             constant_pool_count;
              cp_info        constant_pool[constant_pool_count-1]; //常量池，字面量和符号引用
              u2             access_flags; //访问标志1
              u2             this_class; //全限定名
              u2             super_class; //父类全限定名
              u2             interfaces_count; //接口数量
              u2             interfaces[interfaces_count]; //接口的全限定名
              u2             fields_count;
              field_info     fields[fields_count]; //类或接口的字段
              u2             methods_count;
              method_info    methods[methods_count]; //方法表
              u2             attributes_count;
              attribute_info attributes[attributes_count]; //属性表，code，exception等
}
```

* 无符号数 基本的数据类型，u1,u2,u4,u8类表示1，2，4，8个字节的无符号数，可以描述数字，索引引用，数字量，按UTF-8编码的字符串。
* 表 是由多个无符号数组成的复合数据类型，所有表以_info结尾，整个class就是一张表

### 魔数
Class文件的头4个字节，作用：确定这个文件是否为一个能被虚拟机接受的Class文件。值为：0xCAFEBABE(咖啡宝宝)
### 版本号
紧接着魔数的4个字节。第5，6字节是次版本号，第7，8字节是主版本号
### 常量池
常量池可以理解为class文件的资源仓库。主要存放了两大类常量：**字面量和符号引用**。字面量主要存储文本字符串、声明为final的常量值。而符号引用主要包含以下三类常量:

1. 类和接口的全限定名
2. 字段的名称和描述符  
3. 方法的名称和描述符

**java代码再进行javac编译生成class文件时没有“链接”的步骤。在class文件中不会保存各个方法和字段的最终内存布局信息，在虚拟机加载class文件时才会动态链接。**
常量池的14中常量结构 比如CONSTANT_CLASS_info,代表类或接口的符号引用，表中的name_index指向CONSTANT_UTF8_info类型的数据，这个存放着我们类的全限定名。
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474798237094.jpg)
### 访问标志
常量池结束后的两个字节是表示访问标志，用于识别类或者接口的访问信息，比如是类还是接口，访问类型，是否final等等。
### 类索引，父类索引，接口索引集合
class文件由这三个数据来确定类的继承关系。类索引(this_class)和父类索引(super_class)是u2类型的数据，接口索引集合是u2类型的数据集合。
类索引可以确定类的全限定名，父类索引可以确定类的父类全限定名，除了Object类，其他类都有1个父类。接口索引集合按照implements顺序从左到右排列在集合索引中
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474798810139.jpg)
### 字段表集合
字段表描述类或者接口中申明的变量。字段包括类变量和实例变量，不包括方法内部的局部变量。字段表的格式如下所示，access_flags是字段的访问标志，public，可变性final，并发性violatile等等。其中name_index和descriptor_index是对常量池的引用，代表字段的简单名称和方法的描述符。
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474798607604.jpg)
### 全限定名，简单名称，描述符的区别

1. 全限定名是`org/fenixsoft/clazz/TestClass`是类的全限定名
2. 简单名称 指没有类型和参数修饰符的方法或者字段名称 inc()方法和m字段的简单名称为inc 和m
3. 字段和方法的描述符的解释如下
字段的描述符如下所示
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474799045828.jpg)

对于数组类型，每一维度用`[`表示，比如`java.lang.string[][]`的描述符为[[Ljava/lang/String, int[]的描述符为[I
描述符描述方法时按照先参数列表再返回值，比如void inc()表示为()V 方法`java.lang.String toString()` 表示为()Ljava/lang/String, 方法int indexOf(char[]source, int sourceOffset)可以表示为([CI)I。

### 方法表集合
方法表是class文件中的方法的集合。方法表和字段表类似，结构也是访问标志，名称索引，描述符索引，属性表集合等。
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474799426642.jpg)

方法的定义在可以通过方法的访问标志，名称索引，描述符索引表达清楚，**方法内部的代码是经过java的编译器编译成字节码后存放在方法属性集合中的名为“code”的属性中**

在java语言中，重载(override)一个方法，除了和原方法的简单名称一样，还需要和原方法有一个不同的**特征签名**,特征签名是一个方法中不同参数在常量池中的字段符号引用的合集，所以返回值不会包含在特征签名中，所以**无法通过返回值来重载方法**。
### 属性表
属性表中存放的是java代码的字节码指令的集合，比如方法中的具体执行字节码均是通过
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474799656221.jpg)
#### code属性
**java代码经过javac编译后会变成字节码指令存储在code属性中**。code属性存放在方法表的属性中，但是不是所有的方法表中都存在code属性，比如接口和抽象类的方法就不存在code属性
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474799781956.jpg)

- max_stack是操作数栈深度的最大值
- max_locals是局部变量所需的空间  
max_locals的单位值slot，slot是虚拟机为局部变量分配内存的最小单位，除了double和long两者是需要2个slot存放，其他的都是1个slot来存放。方法参数包括this，异常处理的参数，即try catch中定义的异常，方法体中的局部变量都是用slot来存放。slot可以被复用，只要保证正确性。
- code_length和code是存储的字节码
- exception是方法中可能抛出的受查异常，也就是throws的异常

#### 异常表
编译器是采用异常表而不是简单的跳转命令来实现java的异常和finally处理机制
![](https://medesqure.oss-cn-hangzhou.aliyuncs.com/img/20190624233930.png)

看出0-9行是正常返回，10-20是exception型的异常返回，21-25是非exception得异常返回。**3种路径都有finally中的代码，finally的代码是会嵌套在3种路径的代码之后在return之前**。
结果是没有异常，返回1，出现exception异常，返回是2，出现exception以外的异常，方法没有返回值。
**在return方法执行时，会先把x的值存在returnValue里，饭后finally代码块中执行对x赋值。最后返回的returnValue的值。所以最后代码的返回还是1，而出现异常后对returnValue重新赋值了，所以返回2**。
### 字节码指令简介
* 加载和存储指令
加载和存储指令将数据在栈帧的局部变量表和操作数栈之间传输
* 运算指令
将连个操作数栈的值进行运算，再将结果存回操作数栈顶
* 类型转换
转换类型，虚拟机直接支持从小范围类型向大范围类型的安全转换，大数到小数就要使用转换指令
* 对象创建和访问指令
new，newarray，访问类字段 getstatic 访问非类字段 getfield
* 方法调用指令
    1. invoke virtual 调用方法的虚方法，根据方法的实际类型进行分派
    2. invoke interface 调用接口的方法，搜索实现接口方法的实例对象并找出最合适的方法调用
    3. invoke special 调用特殊的方法，比如实力初始化方法和私有方法和父类方法
    4. invokedynamic 运行时动态解析出引用的方法。这个是用户所设定的引导方法决定的。(用的地方较少？？)
    
### 同步指令
java虚拟机支持方法级的同步和方法内部指令的同步，两种的同步结构都是使用**管程(Monitor)** 来支持。比如`synchronize`的语句实现是monitorenter和monitorexit来实现，执行的线程必须先成功持有管程，然后才能执行方法，方法最后成功或者非正常完成都会释放管程。**同步方法在执行时跑出异常，在内部无法处理异常，同步方法持有的管程在异常被抛到同步方法之外时也被自动释放**。
一个方法是不是同步方法是可以通过方法常量池的方法表结构中的是否同步标志得到，当是一个同步方法就必须先获取管程才能继续执行。









