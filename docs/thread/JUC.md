# JUC



JUC 是 java.util.concurrent 包下面的并发工具类的简称。在 JDK 的并发包里提供了几个非常有用的并发工具类。CountDownLatch、CyclicBarrier 和 Semaphore 工具类提供了一种并发流程控制的手段，Exchanger 工具类则提供了在线程间交换数据的一种手段。

# CountDownLatch

countdownlatch 是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完毕再执行。从命名可以解读到 countdown 是倒数的意思，类似于我们倒计时的概念。

模拟投票场景：

```java
public static void main(String[] args) throws InterruptedException {
    int size = 10; //定义十个人投票
    CountDownLatch latch = new CountDownLatch(size); //定义10个容量计数器
    for (int i = 0; i < size; i++) {
        // 模拟投票
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "  投票...");
            latch.countDown(); //投完票计数器减一
        }).start();
    }
    latch.await();//等待计数器为0 才进行下面的任务
    System.out.println("所有人投票完毕，统计票数...");
}
```

CountDownLatch的api：

1. CountDownLatch.await() 使当前线程在锁存器倒计数至零之前一直等待，除非线程被中断或超出了指定的等待时间。
2. CountDownLatch.countDown() 递减锁存器的计数，如果计数到达零，则释放所有等待的线程。
3. CountDownLatch.getCount() 返回当前计数。
4. CountDownLatch.toString() 返回标识此锁存器及其状态的字符串。状态用括号括起来，包括字符串 "Count ="，后跟当前计数。

# CyclicBarrier

CyclicBarrier 就是可循环使用的屏障。它让一组线程到达一个屏障（同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会打开，所有被屏障阻拦的线程才会继续执行。

简单来说，就是定义n个线程，当线程处理完，就会等待，直到所有线程处理完，才会进行下一步操作或者结束。

模拟开会场景

```java
public class CyclicBarrierDemo {

    Random random = new Random();
    public void meet(CyclicBarrier cyclicBarrier){
        try {
            TimeUnit.SECONDS.sleep(random.nextInt(5)); // 假设去会议室的时间
            System.out.println(Thread.currentThread().getName() + " 去会议室....");
            cyclicBarrier.await(); // 等待其他人开会
            System.out.println(Thread.currentThread().getName() + " 散会");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) throws Exception {
        CyclicBarrierDemo d = new CyclicBarrierDemo();
        int size = 3; //定义队员数量
        //定义屏障数量，并达到数量之后完成对应的操作
        CyclicBarrier barrier = new CyclicBarrier(size, () -> {
            System.out.println("会议中....");
            try {
                TimeUnit.SECONDS.sleep(4);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        for (int i = 0; i < size; i++) {
            new Thread(() -> d.meet(barrier)).start();
        }
    }

}
```



CyclicBarrier的api：

1. CyclicBarrier.await() 在所有参与者都已经在此 barrier 上调用 await 方法之前，将一直等待。
2. CyclicBarrier.getNumberWaiting() 返回当前在屏障处等待的参与者数目。
3. CyclicBarrier.getParties() 返回要求启动此 barrier 的参与者数目。
4. CyclicBarrier.isBroken() 查询此屏障是否处于损坏状态。
5. CyclicBarrier.reset() 将屏障重置为其初始状态。



**注意点** 

1. 对于指定计数值 `parties`，若由于某种原因，没有足够的线程调用`CyclicBarrier.await()`，则所有调用`await `的线程都会被阻塞； 
2. 同样的`CyclicBarrier`也可以调用`await(timeout, unit)`，设置超时时间，在设定时间内，如果没有足够线程到达，则解除阻塞状态，继续工作； 
3. 通过 reset 重置计数，会使得进入 await 的线程出现 BrokenBarrierException； 
4. 如果采用是 `CyclicBarrier(int parties, Runnable barrierAction)`构造方法，执行 barrierAction 操作的是最后一个到达的线程

# Semaphore



从概念上讲，信号量维护了一个许可集。如有必要，在许可可用前会阻塞每一个 acquire()，然后再获取该许可。每个 release() 添加一个许可，从而可能释放一个正在阻塞的获取者。

实际业务场景运用：限流

简单模拟限流场景代码

```java
public class SemaphoreDemo {
    public void limit(Semaphore semaphore) {
        try {
            semaphore.acquire();// 获取信号量
            // 入参可以传入 HttpServletRequest
            System.out.println(Thread.currentThread().getName()+ " 处理请求");
            TimeUnit.SECONDS.sleep(2);
            semaphore.release();// 释放信号量
            System.out.println(Thread.currentThread().getName()+ " 处理完毕，释放请求");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        // 创建一个容量2的公平的信号量
        Semaphore semaphore = new Semaphore(2, true);
        SemaphoreDemo semaphoreDemo = new SemaphoreDemo();
        //模拟并发请求
        for (int i = 0; i < 100; i++) {
            new Thread(()->{
                semaphoreDemo.limit(semaphore);
            }).start();
        }
    }
}
```



Semaphore的api：

1. Semaphore.acquire() 获取信号量
2. Semaphore.release() 释放信号量







# Exchanger

专业术语：对元素进行配对和交换的线程的同步点。每个线程将条目上的某个方法呈现给 exchange 方法，与伙伴线程进行匹配，并且在返回时接收其伙伴的对象。Exchanger 可能被视为 SynchronousQueue 的双向形式。Exchanger 可能在应用程序（比如遗传算法和管道设计）中很有用。

用途：可以用于两个线程某个对象的比对、或者对象的传递

```java
public class ExchangerDemo {
    public void a(Exchanger<String> exchanger) {
        System.out.println("a 方法执行...");
        try {
            System.out.println("a 线程正在抓取数据...");
            Thread.sleep(2000);
            System.out.println("a 线程抓取到数据...");
            String res = "12345";
            System.out.println("a 等待对比结果...");
            String value = exchanger.exchange(res);
            System.out.println("开始进行比对...");
            System.out.println("比对结果为：" + value.equals(res));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
    public void b(Exchanger<String> exchanger) {
        System.out.println("b 方法开始执行...");
        try {
            System.out.println("b 方法开始抓取数据...");
            Thread.sleep(4000);
            System.out.println("b 方法抓取数据结束...");
            String res = "12345";
            exchanger.exchange(res);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        ExchangerDemo d = new ExchangerDemo();
        Exchanger<String> exchanger = new Exchanger<>();
        new Thread(() -> d.a(exchanger)).start();
        new Thread(() -> d.b(exchanger)).start();
    }
}
```



Exchanger的api：

1. Exchanger.exchange() 等待另一个线程到达此交换点（除非当前线程被中断），然后将给定的对象传送给该线程，并接收该线程的对象。
   如果另一个线程已经在交换点等待，则出于线程调度目的，继续执行此线程，并接收当前线程传入的对象。当前线程立即返回，接收其他线程传递的交换对象。


