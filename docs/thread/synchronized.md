# synchronized

## synchronized 用法

- 普通同步方法，锁是当前实例对象
- 静态同步方法，锁是当前 class 对象
- 同步方法块，锁是括号里面的对象

示例：

```java
public class SynchronizedDemo {
    int count1;
    static int count2; 
    public synchronized void add1(int value) {
        count1 += value;
    }
    public void add2(int value) {
        synchronized (this) {
            count1 += value;
        }
    }
    public static synchronized void add3(int value) {
        count2 += value;
    }
    public static void add4(int value) {
        synchronized (SynchronizedDemo.class) {
            count2 += value;
        }
    }
}
```

## synchronized 原理

大致原理如下：

1. 可以把执行 monitorenter 指令理解为加锁，执行 monitorexit 理解为释放锁。
2. 每个对象维护着一个记录着被锁次数的计数器。
3. 未被锁定的对象的该计数器为0，当一个线程获得锁（执行monitorenter）后，该计数器自增变为1，当同一个线程再次获得该对象的锁的时候，计数器再次自增。当同一个线程释放锁（执行monitorexit指令）的时候，计数器再自减。
4. 当计数器为0的时候。锁将被释放，其他线程便可以获得锁。

**实现细节**

使用 javap 对 class 文件进行反编译后可以看到代码里面插入了monitorenter、monitorexit指令。

```java
public class SynchronizedDemo {
    public static void main(String[] args) {
        synchronized (SynchronizedDemo.class) {
            m();
        }
    }
    public static void m() {
        System.out.println("hello synchronized");
    }
}
```

在 `SynchronizedDemo.class` 同级目录下执行 `java -v SynchronizedDemo` 输出内容中截取部分主要代码。

```java
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc           #2                  // class SynchronizedDemo
         2: dup
         3: astore_1
         4: monitorenter
         5: invokestatic  #3                  // Method m:()V
         8: aload_1
         9: monitorexit
        10: goto          18
        13: astore_2
        14: aload_1
        15: monitorexit
        16: aload_2
        17: athrow
        18: return

```

上面展示 class 信息中，对于同步块的实现使用了 monitorenter 和 monitorexit 指令。

任意一个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取到该对象的监视器才能进入同步块活同步方法，而没有获取到监视器的线程会被阻塞在同步块和同步方法的入口处，进入 BLOCKED 的状态。

![](image/synchronized-3.png ':size=50%')

## synchronized 锁的升级与对比

synchronized 又被称为重量级锁。Java SE 1.6 对 synchronized 进行了优化，引入了偏向锁和轻量级锁。

在 Java SE 1.6 中，锁一共有四种状态，级别从低到高依次是：无所状态、偏向锁状态、轻量级锁状态、重量级锁状态。

锁可以升级但不能降级，目的是为了提高获得锁和释放锁的效率。

### 偏向锁

大多数情况下，锁不仅不存在多线程竞争，而且总是有同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。

**基本原理**

当一个线程访问加了同步锁的代码块时，会在对象头中存储当前线程的 id，后续这个线程进入和退出这段加了同步锁的代码时，不需要再次加锁和释放锁，而是直接比较对象头里是否存储了指向当前线程的偏向锁。如果相等表示偏向锁是偏向于当前线程，就不要去获得锁操作。

**偏向锁的获取逻辑**

1. 首先获取锁对象的 markword，判断是否处于可偏向状态。
2. 如果是可偏向状态，则通过 CAS 操作，把当前线程 id 写入到 markword 中
   - 如果 CAS 成功，表示已经获得锁对象的偏向锁，接着执行同步代码块
   - 如果 CAS 失败，说明有其他线程已经获得了偏向锁，这种情况说明当前锁存在竞争，需要撤销已经获得偏向锁的线程，并升级到轻量级锁
3. 如果是已偏向状态，需要检查 markword 中存储的线程 id 是否等于当前线程 id
   - 如果相等，不要再次获得锁，可直接指向同步代码块
   - 如果不相等，说明当前锁偏向于其他线程，需要撤销偏向锁并升级到轻量级锁

**偏向锁的撤销逻辑**

偏向锁的撤销并不是把对象恢复到无锁可偏向状态（因为偏向锁并不存在锁释放的概念），而是在获取偏向锁的过程中，发现 CAS 失败（存在线程竞争时），直接把偏向的锁对象升级到被加了轻量级锁的状态。

对原持有偏向锁的线程进行撤销时，原获得偏向锁的线程有两种情况：

1. 原获得偏向锁的线程如果已经退出了临界区（同步代码块执行完），那么这个时候会把对象头设置成无锁状态并且争抢锁的线程可以基于 CAS 重新偏向当前线程
2. 如果原获得偏向锁的线程处于临界区（同步代码块未执行完），这个时候会把会的偏向锁的线程升级为轻量级锁后继续执行同步代码块



**流程图**

![](image/synchronized-1.png ':size=50%')

### 轻量级锁

**轻量级锁的加锁逻辑**

锁升级为轻量级锁之后，对象的 markword 也会进行相对的变化。升级为轻量级锁的过程：

1. 线程在自己的栈桢中创建锁记录
2. 将锁对象的对象头中 markword 复制到线程刚刚创建的锁记录中
3. 将锁记录中的 owner 指针指向锁对象（自旋锁操作）
4. 将锁对象的对象头的 markword 替换为锁记录的指针



**轻量级锁的解锁逻辑**

轻量级锁的所释放逻辑其实就是获得锁的逆向逻辑，通过 CAS 操作把线程栈桢中的锁记录替换回锁对象的 markword ，如果成功表示没有竞争，如果失败，表示存在竞争，那么轻量级锁就会升级为重量级锁。



![](image/synchronized-2.png ':size=50%')

### 自旋锁

轻量级锁在加锁过程中用到了自旋锁。

所谓自选，就是指当有另外一个线程来竞争锁时，这个线程会在原地循环等待，而不是把该线程阻塞，直到那个获得锁的线程释放之后，这个线程才可以获得锁。

所以，轻量级锁适用于那些同步代码块执行很快的场景，这样线程原地等待很短的时间就可以获得锁了。

!> 自旋锁在循环的时候，会消耗cpu资源

所以自旋锁必须要有一定的条件控制，否则当同步代码块执行时间很长，会持续消耗 cpu 资源。

### 自适应自旋锁

在 Java 1.6 之后，引入了自适应自旋锁，因为自旋锁的次数不是固定不变的，而是根据前一次在同一个锁上自旋的时间以及锁的拥有者的状态来决定。

通过在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而将允许自旋等待持续相对更长的时间。如果对于某个所，自旋很少成功获得过锁，那在以后尝试获得这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费 cpu 资源。

### 重量级锁

当轻量级锁升级为重量级锁之后，意味着线程只能被挂起阻塞等待被唤醒了。

通过 monitor 对象来实现重量级锁。





