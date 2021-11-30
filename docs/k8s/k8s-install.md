# k8s kubeadm搭建



## 前置准备

- 三台 centos 7+ 的机器集群 
- 配置大于 2G 内存 2核 CPU
- 集群网络互通，可以联通外网
- 集群节点不能有重复的主机名、MAC、product_uuid
- 关闭防火墙等限制（生产自己注意）
- 禁用 swap



## 更新安装必要的 yum 资源

```sh
yum -y update
yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
```



## 安装 docker

[官方安装说明](https://docs.docker.com/engine/install/centos/#install-docker-engine)

```sh
# 安装 docker 需要的依赖
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# 设置 docker 仓库源地址
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 设置镜像加速器
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["这边替换成自己的实际地址"]
}
EOF
sudo systemctl daemon-reload

# 安装 docker
yum install -y docker-ce docker-ce-cli containerd.io

# 启动并设置开机启动
sudo systemctl start docker && sudo systemctl enable docker
```



## 修改 hosts 文件

**主节点：**

```sh
# 设置注解的hostname，并且修改hosts文件
sudo hostnamectl set-hostname m

vi /etc/hosts
192.168.3.20 m
192.168.3.21 w1
192.168.3.22 w2
```

**工作节点：**

```sh
# 分别设置两个工作节点的 hostname，并且修改hosts文件
sudo hostnamectl set-hostname w1
sudo hostnamectl set-hostname w2

vi /etc/hosts
192.168.3.20 m
192.168.3.21 w1
192.168.3.22 w2
```



> 配置完成之后三个节点互相 ping 一下，查看网络是否可以使用。



## 系统前提配置

1. 官方说明
2. 关闭selinux
3. 关闭 swap
4. 配置 iptables
5. [允许 iptables 检查桥接流量](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#letting-iptables-see-bridged-traffic)

```sh
# (1)关闭防火墙
systemctl stop firewalld && systemctl disable firewalld

# (2)关闭selinux
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# (3)关闭swap
swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab

# (4)配置iptables的ACCEPT规则
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT

# (5)允许 iptables 检查桥接流量 
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```



## 安装 kubeadm、kubelet and kubectl

### 配置 kubernetes.repo

```sh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```



### 安装 kubelet and kubeadm & kubectl

```sh
yum -y install kubelet kubeadm kubectl
```



### 将 docker 和 k8s 设置成同一个 cgroup

- [为什么要设置 cgroup](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#configuring-a-cgroup-driver) 
- [设置 docker 的 cgroup](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker) 



```sh

# 修改 docker 配置信息
vi /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
# 重启 docker
systemctl restart docker
    
# kubelet，这边如果发现输出directory not exist，也说明是没问题的
sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# 重启 kubelet
systemctl enable kubelet && systemctl start kubelet
```



## 拉取 k8s 必需的镜像

### 查看kubeadm使用的镜像

执行 `kubeadm config images list` 查看 kubeadm 需要使用的镜像  [官方离线安装说明](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#running-kubeadm-without-an-internet-connection) 

### 拉取镜像

由于网络限制，不可以直接拉取上述所使用的镜像

可以先拉取阿里同步的镜像，然后 docker tag xxx  new-xxx 来变成 k8s 所需要的镜像

比如操作如下步骤

```sh
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.1 k8s.gcr.io:v1.21.1
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.1
```

操作完所有 kubeadm 需要使用的镜像



## kubeadm init 初始化 master

### 在主节点执行并保存输出的日志信息

- kubernetes-version: k8s 的版本
- apiserver-advertise-address: 主节点的ip 地址

```sh
kubeadm init --kubernetes-version=1.21.1 --apiserver-advertise-address=192.168.3.20 --pod-network-cidr=10.244.0.0/16
```



### 执行日志内容

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**此时kubectl cluster-info查看一下是否成功**



### 验证 pod 是否启动成功

等待一会儿，输入 `kubectl get pods -n kube-system` ，同时可以发现像etcd，controller，scheduler等组件都以pod的方式安装成功了。

**coredns没有启动，需要安装网络插件。**



### 安装网络插件

网络插件有很多 [官方提供的网络插件](https://kubernetes.io/docs/concepts/cluster-administration/addons/)

常用的网络插件

- [calico](https://docs.projectcalico.org/getting-started/kubernetes/) 
- [flannel](https://github.com/flannel-io/flannel#deploying-flannel-manually) 

这里我们选择 `calico` 作为网络插件。 [calico 安装说明](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-50-nodes-or-less)

```sh
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

等待 `calico` 的 pod 都启动成功。



## kubeadm join 加入 node 节点

在 `kubeadm init` 成功之后输入的一大堆日志信息中，有一段如下的日志：（实例）

```sh
kubeadm join 192.168.3.20:6443 --token yu1ak0.2dcecvmpozsy8loh \
    --discovery-token-ca-cert-hash sha256:5c4a69b3bb05b81b675db5559b0e4d7972f1d0a61195f217161522f464c307b0
```

在两个 worker 同时执行上面  `kubeadm join` 命令。

回到 master 节点检查集群信息，查看集群是否部署完成。node 的状态都是 `READY` 代表成功。

```sh
kubectl get nodes
```



## 简单查看节点是否搭建成功



## 新建 nginx_pod.yaml

```yaml
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
```



### 创建 nginx pod

```sh
kubectl apply -f nginx_pod.yaml
```



### 查看 pod 

```sh
kubectl get pods -o wide

# 输出如下
NAME       READY     STATUS   RESTARTS   AGE             IP             NODE   
nginx-pod   1/1     Running      0       40m       192.168.80.194        w2 
```



## 总结

这样 k8s 集群就搭建完了。

从上述步骤来看，最麻烦的就是拉取镜像，因为网络问题必须要转一遍。







