# JVM 调优

要针对 JVM 调优，那必须知道 JVM 的设置参数代表的内容，以及 JVM 常用命令，JVM 分析工具。

# JVM 参数

JVM 参数主要有三大类标准参数、-X参数、-XX参数

## 标准参数

一些不会通用参数，各JDK版本之间通用

```bash
java -version
java -cp
java -help
```

## -X 参数

非标准参数，随着JDK版本变化而变化

```bash
java -Xint -version #解释执行
java -Xcomp -version #第一次使用就编译成本地代码 
java -Xmixed -version #混合模式，JVM自己来决定
```

## -XX 参数

使用最多的参数类型，主要用于 JVM 调优和Debug

```java
a. Boolean 类型
    格式： -XX:[+/-]<name>   +/- 表示是否启用
    例子： -XX:+UseConcMarkSweepGC   表示启用CMS类型的垃圾回收器
b. 非 Boolean 类型
    格式: -XX<name>=<value>    表示name属性的值是value 
    例子: -XX:MaxGCPauseMillis=500
```



## 其他类型

```bash
-Xms1000等价于-XX:InitialHeapSize=1000 
-Xmx1000等价于-XX:MaxHeapSize=1000 
-Xss100等价于-XX:ThreadStackSize=100
```



## 常用参数含义



| 参数                                                         | 含义                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| -XX:CICompilerCount=3                                        | 最大并行编译数                                               | 如果设置大于1，虽然编译速度会提高，但会增加JVM奔溃的可能     |
| -XX:InitialHeapSize=100M                                     | 初始化堆大小                                                 | 简写 -Xms100M                                                |
| -XX:MaxHeapSize=100M                                         | 最大堆大小                                                   | 简写 -Xmx100M                                                |
| -XX:NewSize=20M                                              | 年轻代大小                                                   |                                                              |
| -XX:MaxNewSize=50M                                           | 年轻代最大大小                                               |                                                              |
| -XX:OldSIze=20M                                              | 年老代大小                                                   |                                                              |
| -XX:MetaspaceSize=50M                                        | 方法区最大大小                                               |                                                              |
| -XX:+UseParallelGC                                           | 使用 UseParallel GC                                          | 新生代，吞吐量优先                                           |
| -XX:+UseParallelOldGC                                        | 使用 UseParallelOldGC                                        | 老年代，吞吐量优先                                           |
| -XX:+UseConcMarkSweepGC                                      | 使用CMS                                                      | 老年代，停顿时间优先                                         |
| -XX:+UseG1GC                                                 | 使用G1GC                                                     | 新生代、老年代，停顿时间优先                                 |
| -XX:NewRatio=4                                               | 新老代的比值                                                 | 比如=4，则表示新生代:老年代=1:4,也就是新生代占整个堆内存的五分之一 |
| -XX:SurvivorRatio=8                                          | Survivor和Eden 的比值                                        | 比如=8，表示s0:s01:Eden=1:1:8                                |
| -XX:+HeapDumpOnOutOfMemoryError                              | 启动堆内存溢出打印                                           | 当JVM堆内存溢出时，自动生成dump文件                          |
| -XX:HeapDumpPath=heap.hprof                                  | 指定堆内存溢出打印目录                                       | 在当前目录生成一个heap.hprof文件                             |
| -XX:+PrintGCDetails<br/>-XX:+PrintGCTimeStamps<br/>-XX:+printGCDateStamps | 打印GC日志                                                   | 可以使用不同的垃圾收集器，对比查看GC情况                     |
| -Xss128k                                                     | 设置每个线程的堆栈大小                                       | 3000-5000最佳                                                |
| -XX:MaxTenuringThreshold=6                                   | 提升年老代的最大临界值                                       | 默认是15                                                     |
| -XX:InitiatingHeapOccupancyPercent                           | 启动并发GC周期时堆内存使用占比                               | G1之类的垃圾回收器用它来触发并发GC周期。=0表示一直执行GC循环，默认为45 |
| -XX:G1HeapWastePercent                                       | 允许浪费堆空间占比                                           | 默认10%，如果回收空间小于10%，则不会触发mixedGC              |
| -XX:ConcGCThreads=n                                          | 并发垃圾收集器使用的线程数量                                 | 默认值随JVM允许平台的不同而不同                              |
| -XX:G1MixedGCLiveThresholdPercent=65                         | 混合垃圾回收周期中要包括的就区域设置占用率阈值               | 默认为65%                                                    |
| -XX:G1MixedGCCountTarget=8                                   | 设置标记周期完成后，对存活数据上线为G1MixedGCLiveThresholdPercent的旧区域执行混合垃圾回收的目标次数 | 默认8次混合垃圾回收，混合回收的目标是要控制次目标次数内      |
| -XX:G1OldCSetRegionThresholdPercent=1                        | 描述MixedGC时，Old region 被加入到CSet中                     | 默认情况下，G1只把10%的old region加入到CSet中                |



# 常用命令

## jps

查看 java 进程

```bash
jps
jps -l
```



## jinfo

实时查看和调整JVM配置参数



## jstat

查看虚拟机性能统计信息





## jstack

查看线程堆栈信息





## jmap





