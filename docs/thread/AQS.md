## AQS

AQS （AbstractQueueSynchronizer，队列同步器 ），是用来构建锁或者其他同步组件的基础框架。它使用一个 int 类型的成员变量表示同步状态，通过内置的 FIFO 队列来完成资源获取线程的排队工作。

## 接口及示例

**AQS 可重写的方法**

| 方法名称                                    | 说明                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| protected boolean tryAcquire(int arg)       | 独占式获取同步状态，实现该方法需要查询当前状态并判断同步状态十分符合预期，然后在进行 CAS 设置同步状态 |
| protected boolean tryRelease(int arg)       | 独占式释放同步状态，等待获取同步状态的线程将有机会获取同步状态 |
| protected int tryAcquireShared(int arg)     | 共享式获取同步状态，返回大于等于0的值，表示获取成功，反之，获取失败 |
| protected boolean tryReleaseShared(int arg) | 共享式释放同步状态                                           |
| protected boolean isHeldExclusively()       | 当前同步器释放在独占模式下被线程占用，一般该方法表示是否被当前线程所占用 |



**AQS 提供的模板方法**

| 方法名称                                           | 说明                                                         |
| -------------------------------------------------- | ------------------------------------------------------------ |
| void acquire(int arg)                              | 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法将会调用重写的 tryAcquire(int arg) 方法 |
| void acquireInterruptibly(int arg)                 | 与 acquire() 相同，但是该方法响应中断，当前线程未获取到同步状态而进入到同步队列中，如果当前线程被中断，则该方法会抛出 InterruptedException 并返回 |
| boolean tryAcquireNanos(int arg,long nanos)        | 在 acquireInterruptibly() 基础上增加了超时限制，如果当前线程在超时时间内没有获取到同步状态，那么将会返回 false，如果获取到了返回 true |
| void acquireShared(int arg)                        | 共享锁的获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占锁获取获取的主要区别是在同一时刻可以有多个线程获取到同步状态 |
| void acquireSharedInterruptibly(int arg)           | 与 acquireShared() 相同，该方法响应中断                      |
| boolean tryAcquireSharedNanos(int arg, long nanos) | 在 acquireSharedInterruptibly() 基础上增加了超时限制         |
| boolean release(int arg)                           | 独占锁的释放同步状态，该方法还在释放同步状态之后，将同步队列中第一个接地那包含的线程唤醒 |
| boolean releaseShared(int arg)                     | 共享锁的释放同步状态                                         |
| Collection&lt;Thread&gt; getQueuedThreads()        | 获取等待在同步队列上的线程集合                               |



**独占锁例子**

独占锁就是在同一时刻只能有一个线程获取到锁，而其他获取锁的线程只能处于同步队列中等待，只有获取锁的线程释放了锁，后继的线程才能够获取锁。

```java
public class Mutex implements Lock {

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        //是否处于占用状态
        protected boolean isHeldExclusively() { return getState() == 1; }
        @Override
        // 当状态为0的时候获取锁
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        @Override
        // 释放锁
        protected boolean tryRelease(int arg) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        Condition newCondition() { return new ConditionObject(); }
    }
    private final Sync sync = new Sync();
    public void lock() {sync.acquire(1);}
    public boolean tryLock() {return sync.tryAcquire(1); }
    public void unlock() {sync.release(1);}
    public Condition newCondition() { return sync.newCondition(); }
    public boolean isLocked() { return sync.isHeldExclusively(); }
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    public void lockInterruptibly() throws InterruptedException { sync.acquireInterruptibly(1); }
    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException { return sync.tryAcquireNanos(1, unit.toNanos(timeout)); }
}
```

## 实现分析

### 同步队列

AQS 依赖内部的同步队列来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。

同步队列中的节点用来报错获取同步状态失败的线程引用、等待状态以及前驱和后继节点，节点的属性类型如下

| 属性类型与名称  | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| int waitStatus  | 等待状态。<br/>包含如下状态<br/>1. CANCELLED=1,由于在同步队列中等待的线程等待超时或者中断，需要从同步队列中取消等待，节点进入该状态将不会变化<br/>2. SIGNAL = -1,后继节点的线程出等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，是后继节点的线程得以运行<br/>3. CONDITION = -2,节点在等待队列中，节点线程等待在 Condition 上，当其他线程对 Condition 调用了 signal() 方法后，该节点将会从等待队列中转移到同步队列中，加入到对同步状态的获取中<br/>4. PROPAGATE = -3,表示下一次共享式同步状态获取将会无条件地被传播下去<br/>5. INITIAL = 0, 初始状态 |
| Node prev       | 前驱节点，当节点加入同步队列时被设置                         |
| Node next       | 后继节点                                                     |
| Node nextWaiter | 等待队列中的后继节点。如果当前节点是共享的，那么这个字段将是一个 SHARED 常量，也就是说节点类型（独占和共享）和等待队列中的后继节点共用一个字段 |
| Thread thread   | 获取同步状态的线程                                           |



### AQS  队列结构

![](image/AQS-1.png ':size=60%')

### AQS 类结构



**属性**

```java
// 属性
privatetransientvolatile Node head;// 同步队列头节点
privatetransientvolatile Node tail;// 同步队列尾节点
privatevolatileint state;// 当前锁的状态：0代表没有被占用，大于0代表锁已被线程占用(锁可以重入，每次重入都+1)
privatetransient Thread exclusiveOwnerThread; // 继承自AbstractOwnableSynchronizer 持有当前锁的线程
```

**方法**

```java
// 锁状态
getState()// 返回同步状态的当前值；
setState(int newState)// 设置当前同步状态；
compareAndSetState(int expect, int update)// 使用CAS设置当前状态，保证状态设置的原子性；
    
// 独占锁
acquire(int arg)// 独占式获取同步状态，如果获取失败则插入同步队列进行等待；
acquireInterruptibly(int arg)// 与acquire(int arg)相同，但是该方法响应中断；
tryAcquireNanos(int arg,long nanos)// 在acquireInterruptibly基础上增加了超时等待功能，在超时时间内没有获得同步状态返回false;
release(int arg)// 独占式释放同步状态，该方法会在释放同步状态之后，将同步队列中头节点的下一个节点包含的线程唤醒；
    
// 共享锁
acquireShared(int arg)// 共享式获取同步状态，与独占式的区别在于同一时刻有多个线程获取同步状态；
acquireSharedInterruptibly(int arg)// 在acquireShared方法基础上增加了能响应中断的功能；
tryAcquireSharedNanos(int arg, long nanosTimeout)// 在acquireSharedInterruptibly基础上增加了超时等待的功能；
releaseShared(int arg)// 共享式释放同步状态；
    
// AQS使用模板方法设计模式
// 模板方法，需要子类实现获取锁/释放锁的方法
tryAcquire(int arg)// 独占式获取同步状态；
tryRelease(int arg)// 独占式释放同步状态；
tryAcquireShared(int arg)// 共享式获取同步状态；
tryReleaseShared(int arg)// 共享式释放同步状态；
```

**内部类**

```java
// 同步队列的节点类
staticfinalclass Node {}
```

**Node 类**

```java
staticfinalclass Node {
    volatile Node prev;// 当前节点/线程的前驱节点
    volatile Node next;// 当前节点/线程的后继节点
    volatile Thread thread;// 每一个节点对应一个线程
    
    volatileint waitStatus;// 节点状态
    staticfinalint CANCELLED =  1;// 节点状态：此线程取消了争抢这个锁
    staticfinalint SIGNAL = -1;// 节点状态：当前node的后继节点对应的线程需要被唤醒(表示后继节点的状态)
    staticfinalint CONDITION = -2;// 节点状态：当前节点进入等待队列中
    staticfinalint PROPAGATE = -3;// 节点状态：表示下一次共享式同步状态获取将会无条件传播下去

    Node nextWaiter;// 共享模式/独占模式
    staticfinal Node SHARED = new Node();// 共享模式
    staticfinal Node EXCLUSIVE = null;// 独占模式
}
```

### 独占式同步状态获取与释放

**抢占锁操作代码分析**

```java
/**
 * 1.当前线程通过tryAcquire()方法竞争锁
 * 2.线程抢到锁，tryAcquire()方法返回true，运行结束
 * 3.线程没有抢到锁，addWaiter()方法将当前线程封装成node加入到同步队列，并将node交由AcquireQueued()处理
 */
public final void acquire(int arg) {
    if (!tryAcquire(arg) && // 模板方法，交由子类实现具体逻辑
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

/**
 * 需要子类实现的抢锁的方法
 * 目前可以理解为通过CAS修改state的值，成功即为抢到锁，返回true；否则返回false。
 */
protected boolean tryAcquire(int arg) {
    thrownew UnsupportedOperationException();
}

/**
 * 1.只有head的后继节点能去抢锁，一旦抢到锁旧head节点从队列中删除，next被置为新head节点。
 * 2.如果node线程没有获取到锁，将node线程挂起。
 * 3.锁释放时head节点的后继节点唤醒，唤醒之后继续for循环抢锁。
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {// 自旋
            /**
             * 1.node的前置节点是head时，可以调用tryAcquire()尝试获取锁，获取锁成功则将node设置为head
             * 2.node线程没有获取到锁，执行下一个if块代码
             *  此时有两种情况：
             *     1. node不是head的后继节点，无资格竞争锁
             *     2. node是head的后继节点，但被其他线程抢占了
             */
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            /**
             * shouldParkAfterFailedAcquire()：通过前置节点的waitStatus来判断是否可以将node节点挂起
             * parkAndCheckInterrupt():将当前线程挂起
             * 1. 如果node前置节点p.waitStatus==Node.SIGNAL(-1)，直接将线程挂起，等待唤醒。
             *		锁释放时会将head节点的后继节点唤醒，唤醒之后继续循环竞争锁。
             * 2. 如果node前置节点p.waitSstatus<=0但是不等于-1，
             *		1)shouldParkAfterFailedAcquire()会将p.waitStatus设置成-1，并返回false
             *		2)进入下一次for循环尝试竞争锁，要是没有获取到锁，当p.waitStatus==-1,会执行第一步操作
             * 3. 如果node前置节点p.waitStatus>0
             *		1)shouldParkAfterFailedAcquire()为node找一个waitStatus<=0的前置节点，并返回false
             *		2)继续循环操作
             */		
            
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

/**
 * 通过前置节点pred的状态waitStatus 来判断是否可以将node节点线程挂起
 * pred.waitStatus==Node.SIGNAL(-1)时，返回true表示可以挂起node线程，否则返回false
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        /*
         * waitStatus>0，表示节点取消了排队
         * 这里检测一下，将不需要排队的线程从队列中删除（因为同步队列中保存的是等锁的线程）
         * 为node找一个waitStatus<=0的前置节点pred
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 此时pred.waitStatus<=0但是不等于-1，那么将pred.waitStatus置为Node.SIGNAL(-1)
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

/**
 * 将当前线程挂起
 * LockSupport.park()挂起当前线程；LockSupport.unpark(thread)唤醒线程thread
 */
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);// 将当前线程挂起
    return Thread.interrupted();
}
```

**加入队列操作代码分析**

```java
/**
 * 1.线程获得锁失败后，封装成 Node 加入同步队列
 * 2.如果队列有tail节点，说明节点不为空，可以直接加入队列
 *  2.1 入队时，通过CAS将node设置成tail。CAS 失败说明其他线程先入队操作，node需要通过enq()方法入队
 * 3.如果队列没有tail节点，说明队列未创建，通过enq()入队
 */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);//抢占锁失败之后，把线程封装成node
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {// 如果存在tail
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {//cas将node设置成tail
            pred.next = node;
            return node;
        }
    }
    enq(node);//如果没有tail，node通过enq()方法加入队列
    return node;
}

/**
 * 1.通过自旋的方式将node加入队列，只有node入队成功才返回，否则一直循环
 * 2.如果队列是空的，初始化head、tail，初始完之后再次循环，将node加入队列
 * 3.node入队时，通过CAS将node设置为tail。CAS失败则一直自旋操作，直到成功加入队列
 */
private Node enq(final Node node) {
    for (;;) {// 自旋，直至入队成功
        Node t = tail;
        if (t == null) { // Must initialize
            // 初始化设置head、tail
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            // CAS 将node 加入到队列
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

**释放锁操作代码分析**

```java
/**
 * 释放锁之后，唤醒head的后继节点next
 */
public final boolean release(int arg) {
    if (tryRelease(arg)) {//子类实现去释放锁
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); // 唤醒node节点的后继节点
        return true;
    }
    return false;
}

/**
 * 唤醒node节点（也就是head）的后继节点
 */
private void unparkSuccessor(Node node) {
    
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    //正常情况s就是node.next节点
    Node s = node.next;
    // 有可能head.next取消了等待（waitStatus==1）
    // 那么就从队尾往前找，找到waitStaus<=0的所有节点中排在最前面的去唤醒
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

### 共享式同步状态获取与释放

共享式获取与独占式获取最主要的区别在于同一时刻释放有多个线程同时获取到同步状态。以文件读写为例，程序对文件进行读操作，那么这一时刻对于该文件的写操作均被阻塞，而读操作可以同时进行。写操作要求对资源的独占式访问，而读操作可以使共享式访问。

![](image/AQS-2.png ':size=50%')

**抢占锁操作源码分析**

```java
/**
 * 1.当前线程通过tryAcquireShared()方法竞争锁
 * 2.返回tryAcquireShared()>=0表示竞争到锁，运行结束
 * 3.返回tryAcquireShared()<0，调用doAcquireShared()加入到同步队列
 */
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0) 
        doAcquireShared(arg);
}
// 参考独占式锁逻辑，逻辑大致类似
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
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

### 独占式超时获取同步状态



```java
/**
 * 1.通过tryAcquire()和doAcquireNanos()竞争锁
 * 2.doAcquireNanos()来判断是否超时
 */
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
/*
 * 主要逻辑与独占式同步竞争锁逻辑相似，只不过多了个超时时间判断以及响应中断操作
 */
private boolean doAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    // 计算超时时间
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
            // 为啥要大于spinForTimeoutThreshold(1000纳秒)
            // 因为在很短的时间内超时等待无法做到十分精准，这时候在进行超时等待反而导致超时不准
            // 所以在小于等于spinForTimeoutThreshold这个时间范围内，进行无条件自旋操作
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

**超时竞争锁示意图**

![](image/AQS-3.png ':size=40%')



## 自定义同步组件

### 共享锁demo

通过前面对 AQS 进行的实现分析，来编写一个自定义同步组件来加深对 AQS 的理解。

设计一个同步工具：该工具在同一时刻，只允许至多两个线程同时访问，超过两个线程的访问将被阻塞。

首先，确定访问模式。该工具能在同一时刻支持多个线程的访问，这显然是共享锁的实现。

其次，定义资源数。该工具在同一时刻允许至多两个线程的同时访问，表明同步资源数为 2，这样可以设置初始状态 status=2，当线程竞争锁成功，status-1；线程释放，status+1。在同步状态变更时，需要使用 CAS() 做原子性保证。

**例子**

```java
public class TwinsLock implements Lock {
    public final static class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) {
            if (count <= 0) throw new IllegalArgumentException();
            setState(count);
        }
        @Override
        protected int tryAcquireShared(int arg) {
            for (; ; ) {
                int current = getState();
                int newCount = current - arg;
                if (newCount < 0 || compareAndSetState(current, newCount)) {
                    return newCount;
                }
            }
        }
        @Override
        protected boolean tryReleaseShared(int arg) {
            for (; ; ) {
                int current = getState();
                int newCount = current + arg;
                if (compareAndSetState(current, newCount)) {
                    return true;
                }
            }
        }
    }
    private final Sync sync = new Sync(2);
    @Override
    public void lock() {
        sync.acquireShared(1);
    }
    @Override
    public void lockInterruptibly() throws InterruptedException {

    }
    @Override
    public boolean tryLock() {
        return false;
    }
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return false;
    }
    @Override
    public void unlock() {
        sync.releaseShared(1);
    }
    @Override
    public Condition newCondition() {
        return null;
    }
}
```

**测试用例**

```java
public class TwinsLockTest {
    @Test
    public void test() {
        final Lock lock = new TwinsLock1();
        class Worker extends Thread {
            @Override
            public void run() {
                while (true) {
                    try {
                        lock.lock();
                        SleepUtils.second(1);
                        System.out.println(Thread.currentThread().getName());
                        SleepUtils.second(1);
                    } finally {
                        lock.unlock();
                    }
                }
            }
        }
        //启动10个线程
        for (int i = 0; i < 10; i++) {
            Worker w = new Worker();
            w.setDaemon(true);
            w.start();
        }
        //每隔一秒换行
        for (int i = 0; i < 1000; i++) {
            SleepUtils.second(1);
            System.out.println();
        }
    }
}
public static class SleepUtils{
    public static final void second(long sec) {
        try {
            TimeUnit.SECONDS.sleep(sec);
        } catch (Exception e) {
            // TODO: handle exception
        }
    }
}
```

### 独占式demo

```java
public class Mutex implements Lock {

    private static final class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        Condition newCondition() {
            return new ConditionObject();
        }
    }

    private final Sync sync = new Sync();

    @Override
    public void lock() { sync.acquire(1); }

    @Override
    public void lockInterruptibly() throws InterruptedException { sync.acquireInterruptibly(1); }

    @Override
    public boolean tryLock() { return sync.tryAcquire(1); }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException { return sync.tryAcquireNanos(1, unit.toNanos(time)); }

    @Override
    public void unlock() { sync.release(1); }

    @Override
    public Condition newCondition() { return sync.newCondition(); }
    public boolean isLocked() { return sync.isHeldExclusively(); }
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    public Collection<Thread> getQueuedThreads() { return sync.getQueuedThreads(); }
    public Collection<Thread> getExclusiveQueuedThreads() { return sync.getExclusiveQueuedThreads(); }
}
```

