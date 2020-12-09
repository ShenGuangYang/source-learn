# JVM 线上问题处理思路
一旦线上出现问题，第一目标就是：尽快恢复服务，消除影响。
然后争取保留现场，再去定位问题、处理问题、解决问题。

## 常见问题1：CPU 爆升的情况
常见原因：
- 频繁 GC
- 线程死锁、阻塞
- 死循环

### 通过 arthas 工具排查
下载 `arthas-boot.jar`，然后用java -jar的方式启动：
```shell
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```
选择对应的端口
使用 `thread -n 5` 查看使用 CPU 前5的线程

## 常见问题2：持续GC

在了解 FullGC 原因之前，先花一点时间回顾下 jvm 的内存相关知识：
内存模型
新 new 的对象放在 Eden 区，当 Eden 区满之后进行一次 MinorGC，并将存活的对象放入 S0；
当下一次 Eden 区满的时候，再次进行 MinorGC，并将存活的对象和 S0 的对象放入S1（S0 和 S1 始终有一个是空的）；
依次循环直到 S0 或者 S1 快满的时候将对象放入 old 区，依次，直到 old 区满进行 FullGC。
jdk1.7 之前 Java 类信息、常量池、静态变量存储在 Perm 永久代，类的原数据和静态变量在类加载的时候放入 Perm 区，类卸载的时候清理；在 1.8 中，MetaSpace 代替 Perm 区，使用本地内存，常量池和静态变量放入堆区，一定程度上解决了在运行时生成或加载大量类造成的 FullGC，如反射、代理、groovy 等。

接前面的内容，这个情况下，我们可以去查看gc 的具体情况。
- 查看gc 日志
- 如果所在公司有对应用进行监控的组件当然更方便（比如Prometheus + Grafana）

### GC 原因及定位
1. prommotion failed

从S区晋升的对象在老年代也放不下导致 FullGC（fgc 回收无效则抛 OOM）。

1）survivor 区太小，对象过早进入老年代。

- jstat -gcutil pid 1000 观察内存运行情况；
- jinfo pid 查看 SurvivorRatio 参数；
2）大对象分配，没有足够的内存。

- 日志查找关键字 “allocating large”；
- profiler 查看内存概况大对象分布；
3）old 区存在大量对象。

- 实例数量前十的类：jmap -histo pid | sort -n -r -k 2 | head -10
- 实例容量前十的类：jmap -histo pid | sort -n -r -k 3 | head -10
- dump 堆，profiler 分析对象占用情况
2. concurrent mode failed

在 CMS GC 过程中业务线程将对象放入老年代（并发收集的特点）内存不足。详细原因：

1）fgc 触发比例过大，导致老年代占用过多，并发收集时用户线程持续产生对象导致达到触发 FGC 比例。
- jinfo 查看 CMSInitiatingOccupancyFraction 参数，一般 70~80 即可
2）老年代存在内存碎片。
- jinfo 查看 UseCMSCompactAtFullCollection 参数，在 FullGC 后整理内存


## 线程池异常

Java 线程池以有界队列的线程池为例，当新任务提交时，如果运行的线程少于 corePoolSize，则创建新线程来处理请求。如果正在运行的线程数等于 corePoolSize 时，则新任务被添加到队列中，直到队列满。当队列满了后，会继续开辟新线程来处理任务，但不超过 maximumPoolSize。当任务队列满了并且已开辟了最大线程数，此时又来了新任务，ThreadPoolExecutor 会拒绝服务。

常见问题和原因：
这种线程池异常，一般有以下几种原因：
1. 下游服务响应时间过长
    这种情况有可能是因为下游服务异常导致的，作为消费者我们要设置合适的超时时间和熔断降级机制。
    另外针对这种情况，一般都要有对应的监控机制：比如日志监控、metrics监控告警等，不要等到目标用户感觉到异常，从外部反映进来问题才去看日志查。

2. 数据库慢sql 或者数据库死锁
 查看日志关键字
 获取查日志分析软件
