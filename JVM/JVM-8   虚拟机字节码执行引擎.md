---
title: 深入理解JVM虚拟机 虚拟机字节码执行引擎
date: 2018-08-14 00:38:34

tags: 
  - JVM原理 
  - 虚拟机字节码执行引擎
categories: JVM虚拟机原理
image: https://medesqure.oss-cn-hangzhou.aliyuncs.com/flat-landscape-minimalism-deer-lake-mountains-digital-art-others-20417.jpg

---

# 虚拟机字节码执行引擎

> JAVA虚拟机规范中规定了虚拟机字节码执行引擎的概念模型。从概念模型的角度讲解虚拟机的方法调用和字节码执行。虚拟机的执行引擎的输入都是字节码文件，处理的过程均是字节码解析的等效过程，输出的是执行结果。
<!-- more -->

## 运行时栈帧结构
栈帧是用于支持虚拟机进行**方法调用和方法执行**的数据结构，是虚拟机运行时的**虚拟机栈**的栈元素。栈帧中保存了方法调用的**局部变量表，操作数栈，动态链接和方法返回地址**等信息。每一个方法的开始执行到执行完毕对应着栈帧在虚拟机栈中的入栈和出栈操作。
一个线程中的方法调用链可能很长，在活动线程中，只有栈顶的栈帧才是有效的。
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474804680890.jpg)

### 局部变量表
局部变量表是一组变量值存储空间，主要来保存方法参数和方法内定义的局部变量。在java代码编译时确定了该方法所需分配的局部变量表的最大容量。局部变量的存储单位是slot，一个Slot可以存放一个32位以内（boolean、byte、char、short、int、float、reference和returnAddress）的数据类型，reference类型表示一个对象实例的引用，returnAddress已经很少见了，可以忽略。对于64位的数据类型，long和double采用高位对齐的方式分配连续的两个slot。

使用reference类型表示对一个对象实例的引用，该引用有2个用处。

1. 使用该引用找到java堆中该对象的存储的地址索引。
2. 使用该引用可以找到该对象的class对象元信息，即所属类的数据类型在方法区中存储的类型信息。class对象比较特殊，存储在方法区中。(对象头里面有class地址的信息)

java虚拟机通过索引定位的方式来使用局部变量表，索引范围是从0到最大值。0代表的是this指针。索引n代表使用了第n个slot，64位的数据是连续使用n和n+1的slot。slot可以被复用，为了节省空间。
**类变量在类加载的准备阶段赋零值，在类加载的初始化阶段赋初值。但是局部变量不存在系统初始化的阶段，这意味着定义的局部变量必须手动初始化**。

### 操作数栈
操作数栈是一个后进先出的数据结构，当方法执行时，方法中的操作数对应着**入栈和出栈的操作**。即会各种指令向操作数栈中写入和提取内容。
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474805439707.jpg)

在概念模型中，一个活动线程的两个栈帧是相互独立的，但是虚拟机会做优化处理，让下一个栈帧的部分操作数栈和上一个栈帧的局部变量表重叠，可以共享一部分数据，无需额外的数据赋值传递。

### 动态链接
每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用。持有引用是为了支持方法调用过程的动态链接。class文件中的常量池中存在大量的符号引用，**字节码的方法调用指令是以指向常量池中的符号引用作为参数的**。则可以通过字节码迅速定位到具体的方法代码。**一部分符号引用在类加载或者第一次使用时转化成直接引用，这种转化称为静态解析。另一部分在每次运行时才转为直接引用，称为动态链接**。

### 方法返回地址
存放调用该方法的计数器的值。当一个方法开始后有2种方式退出。

1. 遇到return返回，正常退出。
2. 遇到异常，并且这个异常在方法内没有catch，即在方法内部没有匹配的异常处理器，导致方法退出，方法异常退出不会给调用者返回任何值。

**无论哪种方法退出后都会返回该方法调用的位置，正常退出使用计数器的值作为地址，异常退出要通过异常处理表来确定返回地址。**

## 方法调用
方法调用阶段是为了确定调用方法的版本，即具体调用的是哪一个方法，方法调用和方法执行不同，调用阶段不涉及方法内部的运行过程。**在class文件中存储的都只是符号引用而不是具体的内存布局的入口。只有在类加载阶段或者运行期间才能确定方法的直接引用**。

### 方法的解析
在类加载的解析阶段，会将一部分的符号引用转化成直接引用，这个解析成功的前提条件是：方法在程序真正运行前就有一个可以调用的版本，并且这个版本在运行期间不可改变。
java中的static方法，private方法，init方法和父类方法是“编译期可知，运行期不可变的”。可以在类加载期间进行解析。
java虚拟机中提供了5中方法调用字节码

1. invokestatic:调用静态方法
2. invokespecial:调用构造器，私有方法和父类方法
3. invokevirtual:调用所有的虚方法
4. invokeinterface:调用接口方法，会在运行时才确定一个实现此接口的对象
5. invokedynamic：先在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法。前面4种调用方式的分派逻辑固化在JVM虚拟机，而invokedynamic的分派逻辑是用户的引导方法设置的。

其中，invokestatic和invokespecial指令调用的放啊都可以在解析阶段找到唯一调用版本。符合这个条件的有静态方法，私有方法，实例构造器，父类方法。在类解析时会直接将符号引用转化成直接引用，这些方法称为非虚方法，其他为虚方法。

```java
public class MethodInvokeTest {
    
    public static void sayHello(){
        log.info("我是不可改变的，任何方式都无法改变我的结构...");
    }
    // 为了对比说明，我们来看一下一个普通的方法sayHello()
    public void eatApple(){
        log.info("我是可以改变的，子类可以通过继承改变我的结构");
    }
}

public class MethodInvokeExtendsClass extends MethodInvokeTest{
    
    public void eatApple(){
        log.info("我输出我自己的内容...");
    }
    public static void main(String[] args) {
        MethodInvokeExtendsClass methodInvokeExtendsClass = new MethodInvokeExtendsClass(); // invokespecial指令
        methodInvokeExtendsClass.eatApple(); // invokevirtual指令 --输出：我输出我自己的内容...
        MethodInvokeTest.sayHello();  // invokestatic指令  --我是不可改变的，任何方式都无法改变我的结构...
    }
}
```

### 方法的分派
方法的解析是一个静态的过程，**在编译期间可以得到最终的版本**，在类加载的解析阶段可以把涉及的符号引用全部转化成直接饮用，而方法的分派(Dispacher)调用可能是静态的也可能是动态的。
#### 静态 Dispather
静态分派我理解的最常见的就是方法的重载(overload)了。即在编译期间就要确定是调用哪一个方法。

在方法的重载时是通过参数的静态类型而不是实际类型作为判定依据的。静态类型在编译期间可知的。因此在编译期间，javac可以根据参数的静态类型确定调用那个重载版本。jiyu和liyu的静态类型都是Fish。并且静态分配时发生在编译期间，会自动寻找最合适的函数绑定。
具体代码如下：

```java

public class StaticDispatchTest extends Object{
    final static Log log = LogFactory.getLog(StaticDispatchTest.class);
    // 父类
    static abstract class Fish{}
    // 子类：鲫鱼
    static class Jiyu extends Fish{}
    // 子类：鲤鱼
    static class Liyu extends Fish{}
    
    // 下面写几个重载的方法
    public void swimming(Fish fish){
        log.info("我是鱼，我用鱼鳍游泳...");
    }
    public void swimming(Jiyu jiyu){
        log.info("我是鲫鱼，我用鱼鳍游泳...");
    }
    public void swimming(Liyu liyu){
        log.info("我是鲤鱼，我用鱼鳍游泳...");
    }
    
    // 测试
    public static void main(String[] yangcq){
        Fish jiyu = new Jiyu();
        Fish liyu = new Liyu();
        StaticDispatchTest staticDispatchTest = new StaticDispatchTest();
        staticDispatchTest.swimming(jiyu);  // 打印：我是鱼，我用鱼鳍游泳...
        staticDispatchTest.swimming(liyu);  // 打印：我是鱼，我用鱼鳍游泳...
    }
}
```

#### 动态Dispather
动态Dispather的主要体现时在重写(override)上。
正常的方法调用在调用invokeVirtual方法，invokeVirtual方法的解析

1. 找到操作数栈顶的第一个元素所指向的对象的实际类型，记做C；
2. 在类型C中找到与常量描述符相同的类型和方法，直接通过offset来找到。然后返回该方法的直接引用地址。
3. 如果在C中没有该方法，就从C的父类中去查找，直到找到具体调用的方法，如果没有找到，就抛出异常。

由于invokevirtual指令执行的第一步就是在运行期间确定接收者的实际类型，所以2次调用中的invokevirtual指令把常量池中的类方法的符号引用解析到了不同的直接引用上，这个过程就是Java语言中方法重写(override)的本质。我们把这种运行期间根据实际类型确定方法执行版本的分派过程称为动态分派。

代码如下

```java
public class DynamicDispatchTest extends Object{
    // 父类
    static abstract class Fish{
        public void swimming(){
            System.out.println("我是鱼，我用鱼鳍游泳...");
        }
    }
    // 子类：鲫鱼
    static class Jiyu extends Fish{
        public void swimming(){
            System.out.println("我是鲫鱼，我用鱼鳍游泳...");
        }
    }
    // 子类：鲤鱼
    static class Liyu extends Fish{
        public void swimming(){
            System.out.println("我是鲤鱼，我用鱼鳍游泳...");
        }
    }
    public static void main(String[] yangcq){
        Fish jiyu = new Jiyu();
        Fish liyu = new Liyu();
        jiyu.swimming();  // 打印：我是鲫鱼，我用鱼鳍游泳...
        liyu.swimming();  // 打印：我是鲤鱼，我用鱼鳍游泳...
    }
}
```


#### 虚拟机动态分派的实现
动态分派的实现是在类加载的时候在方法区中建立一个虚方法表， 用虚方法表来代替元数据的查找来提高性能。
虚方法表中存放的是各个方法的实际入口地址。如果方法没有被重写，就和父类的入口地址一样，否则用子类自己的地址。同时，相同的签名的方法，在父类和子类的虚方法表中应当有相同的索引序号，这样在类型转换时候可以方便改变查找的虚函数表。**虚方法表一般在类加载的准备阶段初始化**。

![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474806306668.jpg)

#### java使用Reflection和MethodHandle的区别
在JDK7中新增MethodHandle类，提供一种新的动态确定目标方法的机制。
下面是反射和MethodHandle的方式调用方法的示例。

```java
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
import java.lang.reflect.Method;

public class MethodHandleTest {

    public void printMessage(){
        System.out.println("hello, world");
    }

    public static void main(String[] args) throws Throwable {
        //1.Reflection
        Class clazz = MethodHandleTest.class;
        Object obj = clazz.newInstance();
        Method method = clazz.getMethod("printMessage");
        method.invoke(obj);
        //2.MethodHandle
        MethodHandleTest mht = new MethodHandleTest();
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        MethodHandle methodHandle = lookup.findSpecial(MethodHandleTest.class, "printMessage", MethodType.methodType(void.class), MethodHandleTest.class);
        // MethodHandle methodHandle = lookup.findVirtual(MethodHandleTest.class, "printMessage", MethodType.methodType(void.class));
        methodHandle.invokeExact(mht);
    }
}
```

以下区别：

1. 本质上讲，Relection和MethodHandle机制都是在模拟方法调用，但Reflection是在模拟Java代码层次的方法调用，而MethodHandle是在模拟字节码层次的方法调用。在MethodHandles.lookup中的3个方法-findStaitc、findVirtual、findSpecial正是为了对应于invokestatic、invokevirtual和invokespecial这几条字节码指令的执行权限校验行为，而这些底层细节在使用Reflection API时是不需要关心的。
2. Reflection中的java.lang.reflect.Method对象远比MethodHandle机制信息含量多，换句话说，Reflection是重量级，而MethodHandle是轻量级。
3. 由于MethodHandle是对字节码的方法指令调用的模拟，所以理论上虚拟机做了各种优化。
4. Reflection API的设计目标是只为Java语言服务的，而MethodHandle则设计成可服务于所有Java虚拟机之上的语言，其中也包括Java。





