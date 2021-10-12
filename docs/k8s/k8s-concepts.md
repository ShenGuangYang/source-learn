# k8s 组件介绍

## [Container](https://kubernetes.io/docs/concepts/containers/)

Container 要区分 docker 和 k8s 中的区别。

docker: 镜像的运行时实例为容器

k8s: pod 中运行的容器（可以运行多个 Container）



## [Pod](https://kubernetes.io/docs/concepts/workloads/pods/)

`官网:`  *Pod* 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。

*Pod* 含有（一个或多个）容器（Container）。但 *pod* 基本只含有一个容器，通过 replica 来控制多个服务。

### Pod 初体验

1. **创建 nginx_pod.yaml 文件**

```sh
cat <<EOF> nginx_pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
EOF
```

2. **执行创建 pod 命令**

```sh
kubectl apply -f nginx_pod.yaml
```

3. **查看 Pod 是否启动成功**

```sh
# 查看所有 pod
kubectl get pods 
# 查看所有 pod 并提供更多信息
kubectl get pods -o wide
```

4. **查看 pod 的描述信息**

```sh
 kubectl describe pod nginx-pod-phhrg
```

5. **删除pod**

```sh
kubectl delete pod nginx-pod
# 或者
kubectl delete -f nginx_pod.yaml
```



### Pod yaml 各内容说明

```yaml
# yaml格式对于Pod的定义：
apiVersion: v1          #必写，版本号，比如v1
kind: Pod               #必写，类型，比如Pod
metadata:               #必写，元数据
  name: nginx           #必写，表示pod名称
  namespace: default    #表示pod名称属于的命名空间
  labels:
    app: nginx                  #自定义标签名字
spec:                           #必写，pod中容器的详细定义
  containers:                   #必写，pod中容器列表
  - name: nginx                 #必写，容器名称
    image: nginx                #必写，容器的镜像名称
    ports:
    - containerPort: 80         #表示容器的端口
```



## [ReplcationController(RC)](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicationcontroller/)

`官网:`  *ReplicationController* 确保在任何时候都有特定数量的 *Pod* 副本处于运行状态。 换句话说，*ReplicationController* 确保一个 *Pod* 或一组同类的 *Pod* 总是可用的。

所以，*ReplicationController* 定义了一个期望的场景，即声明某种 *Pod* 的副本数量在任意时刻都符合某个预期值，所以 *ReplicationController* 的定义包含以下几个部分：

- Pod期待的副本数（replicas）
- 用于筛选目标Pod的Label Selector
- 当Pod的副本数量小于预期数量时，用于创建新Pod的Pod模板（template）

也就是说通过RC实现了集群中Pod的高可用，减少了传统IT环境中手工运维的工作。（包括服务高可用、服务宕机自动重启等）



### ReplicationController 初体验

1. **创建 nginx_rc.yaml 文件**

```sh
cat <<EOF> nginx_rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```

2. **执行创建 pod 命令**

```sh
kubectl apply -f nginx_rc.yaml
```

3. **查看pod是否启动成功**

```sh
# 查看所有 pod
kubectl get pods 
# 查看所有 pod 并提供更多信息
kubectl get pods -o wide
# 查看 rc
kubectl get rc
```

4. **尝试删除一个pod，并查看是否会维持副本数的 pod**

```sh
kubectl delete pods nginx-zzwzl
kubectl get pods
```

5. **对pod进行扩缩容**

```sh
kubectl scale rc nginx --replicas=5
```

6. **删除 RC**

```sh
kubectl delete -f nginx_rc.yaml
# 或者
kubectl delete rc nginx
```



### ReplicationController yaml 各内容说明

```yaml
# yaml格式对于rc的定义
apiVersion: v1					 #必写，版本号
kind: ReplicationController      #必写，类型
metadata:                        #必写，元数据
  name: nginx                    #必写，表示rc名称
spec:                            #必写，表示rc的详细定义
  replicas: 3                    #表示受此RC管理的Pod需要运行的副本数
  selector:                      #表示需要管理的Pod的label
    app: nginx
  template:                      #表示用于定义Pod的模板
    # 以下都是 pod 的相关信息
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```



## [ReplicationSet(RS)](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/)

`官网:`  ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。 因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。

v1.2时，RC 就升级成了另外一个概念：Replica Set，官方解释为“下一代RC”。



### ReplicaSet 初体验



1. **创建 nginx_rs.yaml 文件**

```sh
cat <<EOF> nginx_rs.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template: 
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```

2. **执行创建 pod 命令**

```sh
kubectl apply -f nginx_rs.yaml
```

3. **查看pod是否启动成功**

```sh
# 查看所有 pod
kubectl get pods 
# 查看所有 pod 并提供更多信息
kubectl get pods -o wide
# 查看 rc
kubectl get rs
```

4. **尝试删除一个pod，并查看是否会维持副本数的 pod**

```sh
kubectl delete pods nginx-zzwzl
kubectl get pods
```

5. **对pod进行扩缩容**

```sh
kubectl scale rs nginx --replicas=5
```

6. **删除 RC**

```sh
kubectl delete -f nginx_rs.yaml
# 或者
kubectl delete rs nginx
```



### ReplicaSet yaml 各内容说明

```yaml
# yaml格式对于rs的定义
apiVersion: apps/v1				 #必写，版本号
kind: ReplicaSet                 #必写，类型
metadata:                        #必写，元数据
  name: nginx                    #必写，表示rs名称
  labels:                        #表示rs的标签
    app: nginx
spec:                            #必写，表示rc的详细定义
  replicas: 3                    #表示受此Rs管理的Pod需要运行的副本数
  selector:                      #表示需要管理的Pod的label
    matchLabels:
      app: nginx
  template:                       #表示用于定义Pod的模板
  	# 以下都是 pod 的相关信息
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```



### RC 与 RS 如何选择？

RS 是 RC 的升级版本，两者功能相似。但 RS 的功能更加齐全，应当优先选择 RS。 [官网说明](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/#replicationcontroller)

RS 与 RC 唯一的区别是：RS 支持基于集合的 Label Selector（Set-based selector），而 RC 只支持基于等式的 Label Selector（equality-based selector），这使得 Replica Set 的功能更强。



## [Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/)

`官网:`  一个 *Deployment* 为 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 和 [ReplicaSets](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/) 提供声明式的更新能力。

`官网:`  你负责描述 Deployment 中的 *目标状态*，而 Deployment [控制器（Controller）](https://kubernetes.io/zh/docs/concepts/architecture/controller/) 以受控速率更改实际状态， 使其变为期望状态。你可以定义 Deployment 以创建新的 ReplicaSet，或删除现有 Deployment， 并通过新的 Deployment 收养其资源。



### Deployent 初体验



1. **创建 nginx_deployment.yaml 文件**

```sh
cat <<EOF> nginx_deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
EOF
```

2. **执行创建 pod 命令**

```sh
kubectl apply -f nginx_deployment.yaml
```

3. **查看pod是否启动成功**

```sh
# 查看所有 pod
kubectl get pods 
# 查看所有 pod 并提供更多信息
kubectl get pods -o wide
# 查看 rc
kubectl get deployment
```

4. **尝试删除一个pod，并查看是否会维持副本数的 pod**

```sh
kubectl delete pods nginx-deployment-5d59d67564-nhph6
kubectl get pods
```

5. **对pod进行扩缩容**

```sh
kubectl scale rs nginx-deployment-5d59d67564 --replicas=5
```

6. **对 pod 中的 nginx 镜像进行升级**

```sh
kubectl set image deployment nginx-deployment nginx=nginx:1.9.1
```

7. **删除 RC**

```sh
kubectl delete -f nginx_deployment.yaml
# 或者
kubectl delete rs nginx-deployment
```



























