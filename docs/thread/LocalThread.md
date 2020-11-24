# ThreadLocal 

​		一个线程范围的局部变量。

## 简单例子

```java
public class Demo {
    static ThreadLocal<Integer> number = ThreadLocal.withInitial(() -> 0);
    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                Integer nub = number.get();
                nub++;
                number.set(nub);
                System.out.println(Thread.currentThread().getName() + ": " + number.get());
            }, "thread-" + i).start();
        }
    }
}
```



## 实现原理

问题：

1. 每个线程的变量是如何储存的
2. ThreadLocal是什么时候设置初始化的



从使用方法查看源码：

### ThreadLocal.get()



```java
public T get() {
    // 获得当前线程
    Thread t = Thread.currentThread();
    // 根据当前线程获得一个ThreadLocalMap
    // 第一次get()的时候为设置，map为null
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 从ThreadLocalMap取有没有当前线程的值
        ThreadLocalMap.Entry e = map.getEntry(this);
        // 有值就返回对应的值
        // 无值执行下面初始化代码
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 设置初始化值，并返回
    return setInitialValue();
}
```



#### ThreadLocal.getMap(t)

```java
ThreadLocalMap getMap(Thread t) {
    // 返回当前线程中的threadLocals
    return t.threadLocals;
}
```



#### ThreadLocal. ThreadLocalMap.getEntry()

​		ThreadLocalMap 是一个key-value结构的数据结构。



```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```



#### ThreadLocal. setInitialValue()

```java
private T setInitialValue() {
    // 执行初始化方法
    T value = initialValue();
    // 获取当前线程信息
    Thread t = Thread.currentThread();
    // 从线程中获取 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 如果map不为空，重新设置该值
        map.set(this, value);
    else
        // 如果map为空，创建 ThreadLocalMap
        createMap(t, value);
    //返回初始值
    return value;
}
```



#### ThreadLocal.createMap()

```java
// 创建ThreadLocalMap
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

//实例化ThreadLocalMap
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 创建Entry[16]的table
    table = new Entry[INITIAL_CAPACITY];
    // 通过斐波那契散列，来进行table hash槽的下标计算，防止hash冲突
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    // 赋值table[i]
    table[i] = new Entry(firstKey, firstValue);
    // 设置size
    size = 1;
    // 设置容量
    setThreshold(INITIAL_CAPACITY);
}
```





### ThreadLocal.set()



```java
public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 取当前线程的 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 如果map不为空，重新设置该值
        map.set(this, value);
    else
        // 如果map为空，创建 ThreadLocalMap
        createMap(t, value);
}
```



#### ThreadLocal.ThreadLocalMap.set()



```java
private void set(ThreadLocal<?> key, Object value) {
	
    Entry[] tab = table;
    int len = tab.length;
    // 斐波那契散列计算hash槽的下标
    int i = key.threadLocalHashCode & (len-1);
	// 遍历数组
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
		// 相等直接返回
        if (k == key) {
            e.value = value;
            return;
        }
		// key == null 重新设置
        // Entry是弱引用，等于null会被GC回收
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 判断扩容，重新计算hash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```



## 斐波那契散列简单实现



```java
public class Demo {
    private static final int HASH_INCREMENT = 0x61c88647;
    public static void main(String[] args) {
        int size = 16;
        for (int i = 0; i < size; i++) {
            int hashCode = HASH_INCREMENT * i + HASH_INCREMENT;
            System.out.print((hashCode & (size-1)) + " ");
        }
    }
}

// 结果输出 完美分布size中
// 7 14 5 12 3 10 1 8 15 6 13 4 11 2 9 0 
```



