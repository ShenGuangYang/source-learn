# final

本篇文章介绍并发编程中常用的 final 关键字。主要介绍 final 重排序规则。

# final 域的重排序规则

对于 final 域，编译器和处理器要遵守两个重排序规则。

1. 在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
2. 初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。



代码示例：

```java
public class FinalExample {
    int i;                      // 普通变量
    final int j;                // final 变量
    static FinalExample obj;
    public FinalExample() {     // 构造函数
        i = 1;                  // 写普通域
        j = 2;                  // 写final域
    }
    public static void writer() { // 写线程A执行
        obj = new FinalExample();
    }
    public static void reader() { // 读线程B执行
        FinalExample object = obj; // 读对象引用
        int a = object.i;           // 读普通域
        int b = object.j;           // 读final域
    }
}
```

# 写 final 域的重排序规则

写 final 域的重排序规则禁止把 final 域的写重排序到构造函数之外。这个规则的实现包含下面 2 个方面。

1. JMM 禁止编译器把 final 域的写重排序到构造函数之外。
2. 编译器会在 final 域的写之后，构造函数 return 之前，插入一个内存屏障。这个内存屏障禁止处理器把 final 域的写重排序到构造函数之外。

假设线程B读对象引用与读对象的成员域之间没有重排序，下图所示是一种可能的执行时序。

下面执行时序可能会出现：线程A写普通域的操作呗编译器重排序到了构造函数之外，线程B错误地读取了普通变量i初始化之前的值。而写 final 的操作，被写 final 域的重排序规则限定在了构造函数之内，读线程B正确地读取了 final 变量初始化之后的值。

**所以写 final 域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的 final 域已经被正确初始化过了，而普通域不具有这个保障。**

![](image/final-1.png ':size=50%')



# 读 final 域的重排序规则

读 final 域的重排序规则是，在一个线程中，初次读对象引用与初次读该对象包含的 final 域，JMM 禁止处理器重排序这两个操作。编译器会在读 final 域操作的前面插入一个内存屏障。

假设写线程A没有发生任何重排序，下图所示是一种可能的执行时序。

线程B读对象的普通域的操作呗处理器重排序到读对象引用之前。读普通域时，该域还没有被写线程A写入，这是一个错误的读取操作。而读 final 域的重排序会把读对象 final 域的操作限定在读对象引用之后，此时该 final 域已经被A线程初始化过了，这是一个正确地读取操作。

**所以读 final 域的重排序规则可以确保：在读一个对象的 final 域之前，一定会先读包含这个 final 域的对象的引用。**

![](image/final-2.png ':size=50%')



# final 域为引用类型

对于引用类型，写 final 域的重排序规则对编译器和处理器增加如下约束：在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

示例代码

```java
public class FinalReferenceExample {
    final int[] intArrays;                 // final 是引用类型
    static FinalReferenceExample obj;
    public FinalReferenceExample() {       // 构造函数
        intArrays = new int[1];            // 1
        intArrays[0] = 1;                  // 2
    }
    public static void writerOne() {       // 写线程A执行
        obj = new FinalReferenceExample(); // 3
    }
    public static void writerTwo() {       // 写线程B执行
        obj.intArrays[0] = 2;              // 4
    }
    public static void reader() {          // 读线程C执行
        if (obj != null) {                 // 5
            int temp1 = obj.intArrays[0];  // 6
        }
    }
}
```

假设首先线程A执行  `writerOne()` ，执行后线程B执行 `writerTwo()` ，然后线程C执行 `reader()` 。下面是可能的执行时序图：

![](image/final-3.png ':size=50%')



在上图中，1 是对 final 域的写入，2 是对这个 final 域引用的对象的成员域的写入，3 是把构造的对象的引用赋值给某个引用变量。

由写final域的重排序规则“**写final域的操作不能重排序到了构造函数外**”可知， 1 和 3 是不能重排序的。

引用类型 final 域的重排序规则“ **final 引用的对象的成员域的写入不能重排序到了构造函数外**”，保证了 2 和 3 不能重排序。所以线程C至少能看到数组下标 0 的值为 1。

写线程B对数组元素的写入，读线程C不一定能看到。因为写线程B和读线程C之间存在数据竞争，此时的执行结果不可预知。

如果想要确保读线程C看到写线程B对数组元素的写入，写线程B和读线程C直接需要使用 `lock` 或 `volatile` 来确保内存可见性。

