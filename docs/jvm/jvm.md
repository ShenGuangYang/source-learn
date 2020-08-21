# JVM

启动 java 程序之后，会把 java 代码装载到 JVM 中，那么 JVM 是怎么存储 java的代码的呢？

# JVM 运行时数据区



![jvm-1](image\jvm-1.png ':size=50%')



[官方说明文档](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5)

##  Method Area（方法区）

方法区是所有线程共享的内容区域，在虚拟机创建的时候驱动，用于存储已被虚拟机加载的类信息、常量、静态变量、编译器编译后的代码等数据。



## Heap （堆）

堆是虚拟机内存挂管理最大的一块，在虚拟机启动时创建，被所有共享。

Java 对象实例以及数组都是在堆上分配。



## Java Virtual Machine Stacks （虚拟机栈）

虚拟机栈是一个线程执行的区域，保存着一个线程中方法的调用状态。换句话说，一个 Java 线程的运行状态，由一个虚拟机栈来保存，所以虚拟机栈肯定是线程私有的，随着线程的创建而创建。

调用一个方法，就想栈中压入一个栈桢，一个方法执行完成，就会把该栈桢中栈中弹出。



## The pc Register (程序计数器)

程序计数器占用内存空间很少，由于 Java 虚拟机的多线程是通过线程轮流切换，并分配处理器执行时间的方式来实现，在任意时刻，一个处理器只会执行一条线程中的指令。因此，为了线程切换后能够恢复到正确的执行位置，每条线程需要有一个独立的程序计数器。



## Native Method Stacks（本地方法栈）

如果房钱线程执行的方法是 Native 类型，这些方法就会在本地方法栈中执行。



# JVM 内存模型



一块是堆区，一块是非堆区。

堆区分为两大块，一个是 Old 区，一个是 Young 区。

Young 区分为两大块，一个是 Survivor 区（s0,s1）,一个是 Eden 区。Eden:s0:s1 = 8:1:1



![jvm-2](image\jvm-2.png ':size=50%')



## 各个区域理解

我是一个 Java 对象，一开始出生在 Eden 区，有很多兄弟姐妹在 Eden 区中跟我玩。有一天 Eden 区中的人太多了，我就被迫去了 Survivor 区。直到到了 18 岁成年了，就去了 Old 区，在 Old 区生活一段时间，然后被回收了。

![jvm-3](image\jvm-3.png ':size=50%')



# 垃圾回收

## 如何确定一个对象是垃圾

要进行垃圾回收，得先知道什么样的对象是垃圾。

### 引用计数法

对于某个对象而言，只要应用程序中持有该对象的引用，就说明该对象不是垃圾，如果一个对象没有任何指针对其引用，就说明是垃圾。

弊端：如果对象相互引用，就会导致永远不能被回收。

### 可达性分析

通过 GC ROOT 的对象，开始向下寻找，看某个对象是否可达。



## 垃圾回收算法

能够确定对象是垃圾之后，接下来要考虑如何回收。

### 标记-清除（Mark-Sweep）

- 标记：找出内存中需要回收的对象，并且把它们标记出来（会扫描对重所有对象，比较耗时）
- 清除：清除被标记的对象，释放出对应的内存空间



**缺点：**

标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序执行过程中需要分配较大对象时，无法找到足够的连续空间而不得不提前触发另一次内存回收。

### 复制（Copying）

将内存划分成两块相等的区域，每次只使用其中一块。

当其中一块内存使用完了，就将还存活的对象复制到另一块上，然后把已经使用过的内存空间一次清除掉。

**缺点：**

空间利用率低

### 标记-整理（Mark-Compact）

标记整理再试标记清除上的优化，在清除后对内存做整理。



## 分代收集算法

根据对象不同的生命周期将内存划分为不同的区域，针对不同的区域选择不同的算法。

Young 区：复制算法（对象在被分配之后，生命周期可能比较短，Young 区复制效率比较高）

Old 区：标记清除或标记整理 （Old 对象存活时间比较长，整理清除比较好）



# 垃圾回收器



![jvm-4](image\jvm-4.png ':size=50%')



## Serial

Serial 是最基本的垃圾回收器，使用的复制算法。它是一个单线程回收器，它在回收的同时必须暂停其他所有工作线程，直到垃圾回收结束。

## ParNew

ParNew 其实就是 Serial 回收器的多线程版本，也使用的复制算法。它除了多线程回收之外，其他的操作和 Serial 完全一样，在回收的同时也会暂停其他所有工作线程。

## Parallel Scavenge

Parallel Scavenge 回收器也是针对 Young 区的回收器，也使用的复制算法，并使用多线程来处理。它与 ParNew 的区别就是它重点关注于程序达到一个可控制的吞吐量。高吞吐量可以最高效率利用 CPU 来完成程序的运算任务。

## Serial Old

Serial Old 是 Serial 回收器年老代版本，同样是个单线程的收集器，使用标记-整理算法。

## Parallel Old

Parallel Old 是 Parallel Scavenge 的年老代版本，使用多线程的标记-整理算法。

## CMS

CMS (Concurrent mark sweep) 回收器是一种年老代回收器，其最主要目标是获取最短垃圾回收停顿时间。使用的是多线程的标记-清除算法。

## G1

Garbage first 回收器是目前垃圾回收器理论发展的最前沿成果，相比于 CMS 回收器，G1收集器最突出的改进是：

- 基于标记-整理算法，不产生内存碎片。
- 可以精准控制停顿时间，在不牺牲吞吐量前提下，实现低停顿垃圾回收。

G1 回收器避免全区域垃圾收集，它把堆内存划分成大小固定的几个独立区域，并且跟踪这些区域的垃圾回收进度，同时在后台维护一个优先级列表，每次根据所允许的时间，优先回收垃圾最多的区域。



# Java 四大引用类型

## 强引用

Java 中最常见的就是强引用，把一个对象赋值给一个引用变量，这个引用变量就是一个强引用。当一个对象被强引用变量引用时，它处于可达状态，是不可能被垃圾回收机制回收的。

## 软引用

软引用需要用 SoftRefence 类来实现，对于只有软引用的对象来说，当系统内存足够时它不会被回收，当内存不够时会被回收。

## 弱引用

软引用需要用 WeakReference 类来实现，它比软引用的生存期更短，对于只有弱引用的对象来说，只要垃圾回收机制一运行，不管内存空间是否足够，它都会被回收掉。

## 虚引用

虚引用需要 PhantomReference 类来实现，它不能单独使用，必须和引用队列联合使用。虚引用的主要作用就是跟踪对象被垃圾回收的状态。




