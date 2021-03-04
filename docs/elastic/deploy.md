# centos 环境准备
## 关闭防火墙
学习环境关闭防火墙，生成环境打开对应的端口。
```bash
# 查看防火墙状态
systemctl status firewalld
# 关闭防火墙
systemctl stop firewalld
# 开机禁用防火墙
systemctl disable firewalld
```

## 设置 hostname
设置 hostname 可以很清楚的排查es集群的节点的问题。
```bash
# 查看hostname
hostname
# 设置hostname
hostnamectl set-hostname es-01
```

## 禁用内存与硬盘交换

内存与磁盘进行交换，会急剧降低es性能。

在 `/etc/sysctl.conf` 增加以下配置。

```conf
# 禁用内存与硬盘交换
vm.swappiness=1
# 设置虚拟内存大小
vm.max_map_count=262144
```

## 配置文件句柄

在 `/etc/security/limits.conf` 增加以下配置。

```conf
# 进程线程数
*       soft    nproc   131072
*       hard    nproc   131072
# 文件句柄数
*       soft    nofile  131072
*       hard    nofile  131072
# jvm 内存锁定交换
*       soft    memlock unlimited
*       hard    memlock unlimited
```


## 下载 elastic 相关软件

[下载 elastic search](https://www.elastic.co/cn/downloads/)

下载所需要的软件（注意系统版本），比如 elasticsearch、kibana 等。
解压各个软件包。

## 配置 jdk 

解压完 elasticsearch 软件，运行 `./jdk/bin/java -version` 查看 elasticsearch 所需要的 jdk 环境。

下载对应的 jdk 版本。在 `/etc/profile` 配置对应的 jdk 环境变量。

```conf
# 各自的 jdk 解压目录
export JAVA_HOME=/usr/local/java15/jdk-15.0.2
export PATH=$JAVA_HOME/bin:$M2_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```

## 配置 es 账号
因为 es 不能使用 root 账号，所以需要配置一个专门的账号。 es 相关的软件(elasticsearch,kibana等) 都解压到该目录下使用。

```bash
# 添加用户
useradd es
# 修改es目录的用户组及用户
chown -R es:es es/
```

# elastic search 部署

## es 各个配置文件的内容

### config/jvm.options 

jvm 设置原则：
- GC 选择
   - jdk14以上采用G1，以下采用CMS
- 堆栈大小选择
  - 不超过1/2的系统内存
  - 留有1/2的限制内存
  - 配置内存小于32G


```properties
# 内存堆栈大小，不能超过1/2的系统内存
-Xms1g
-Xmx1g

8-13:-XX:+UseConcMarkSweepGC
8-13:-XX:CMSInitiatingOccupancyFraction=75
8-13:-XX:+UseCMSInitiatingOccupancyOnly
14-:-XX:+UseG1GC

# jvm 临时目录
-Djava.io.tmpdir=${ES_TMPDIR}

-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=data
-XX:ErrorFile=logs/hs_err_pid%p.log

## JDK 8 GC logging
8:-XX:+PrintGCDetails
8:-XX:+PrintGCDateStamps
8:-XX:+PrintTenuringDistribution
8:-XX:+PrintGCApplicationStoppedTime
8:-Xloggc:logs/gc.log
8:-XX:+UseGCLogFileRotation
8:-XX:NumberOfGCLogFiles=32
8:-XX:GCLogFileSize=64m
9-:-Xlog:gc*,gc+age=trace,safepoint:file=logs/gc.log:utctime,pid,tags:filecount=32,filesize=64m

```

### config/elasticsearch.yml

```yml
# 集群名称
cluster.name: es-01
# 节点名称,设置成当前集群名称。一台机器上多实例则要重新设置
node.name: ${HOSTNAME}-9200
# 数据目录和日志目录，默认在当前运行程序下，生成环境需要重新指定
path.data: /es/data
path.logs: /es/logs
# 内存交换锁定，需要操作系统设置才能生效
bootstrap.memory_lock: true
# 默认local，仅限本地访问，外网不可以访问，设置 0.0.0.0 通用做法
network.host: 0.0.0.0
# 访问端口
http.port: 9200
transport.port: 9300
# 防止批量删除索引
action.destructive_requires_name: true
# 设置处理器数量，默认无需设置，单机多实例需要设置
node.processors: 4
# 集群模式，默认单机
discovery.type: single-node
# 节点发现
discovery.seed_hosts: ["localhost:9300"]
# 集群初始化节点
cluster.initial_master_nodes: ["localhost:9300"]
# 部分电脑cpu不支持 SSE4.2+，启动会报错，禁用机器学习功能
xpack.ml.enabled: false
```

## jvm 临时目录配置
修改 es 默认的 jvm 的临时目录是挂在 /tmp 目录下，因为 /tmp 目录是linux管理的，有时候会被linux删掉，导致es出现为问题。

两个地方可以设置
- 环境变量中增加 `export ES_TMPDIR=/es/jvm_tmpdir`
- jvm.option 中修改 `-Djava.io.tmpdir=${ES_TMPDIR}` 

# kibana 部署

## kibana 配置文件

```yaml
# 访问端口
server.port: 5601
# 访问地址
server.host: "192.168.0.200"
# es 服务地址
elasticsearch.hosts: ["http://192.168.0.200:9200"]
# kibana元数据存储索引名字，默认 .kabana
kibana.index: ".kibana"
# 国际化
i18n.locale: "en"
```


