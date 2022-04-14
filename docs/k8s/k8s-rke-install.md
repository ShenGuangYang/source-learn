# RKE 搭建 k8s 集群

## 前置准备

- 三台 ubuntu 18+ 服务器
- 配置大于 2G 内存 2核 CPU
- 集群网络互通，可以联通外网
- 集群节点不能有重复的主机名、MAC、product_uuid
- 关闭防火墙等限制（生产自己注意）
- 禁用 swap
- [安装好docker](https://shen33.top/#/k8s/k8s-install?id=%e5%ae%89%e8%a3%85-docker)
- 认真阅读 [官网](https://rancher.com/docs/rke/latest/en/installation/)



## 使用 vagrant 搭建三台Ubuntu服务器

### 创建 Vagrantfile 文件并启动虚拟机

```sh
boxes = [
    {
        :name => "k8s-1",
        :eth1 => "192.168.3.90",
        :mem => "3072",
        :cpu => "2"
    },
    {
        :name => "k8s-2",
        :eth1 => "192.168.3.91",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-3",
        :eth1 => "192.168.3.92",
        :mem => "2048",
        :cpu => "2"
    }
]

ENV["LC_ALL"] = "en_US.UTF-8"

Vagrant.configure(2) do |config|

  config.vm.box = "ubuntu/bionic64"
  
   boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.provider "virtualbox" do |v|
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
          v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
          v.customize ["modifyvm", :id, "--name", opts[:name]]
        end
        # config.vm.network :public_network, ip: opts[:eth1], bridge: "bridge0"
        config.vm.network :public_network, ip: opts[:eth1], bridge: [
          "en7: USB 10/100/1000 LAN"
        ]

        config.vm.provision "shell", run: "always", inline: <<-SHELL
          echo "shenshenshen"
          # yum -y install wget
        SHELL
      end
  end
end
```

### 设置 root 登录(所有虚拟机)

```sh
# 分别登录各个虚拟机设置
# 远程连接到服务器
vagrant shh k8s-1
# 切换 root
sudo -i
# 修改ssh 配置信息
vi /etc/ssh/sshd_config

# 修改一下配置
PermitRootLogin yes
PasswordAuthentication yes

# 重启ssh
service sshd restart
```



## 关闭swap(所有虚拟机)

```sh
swapoff -a # 临时关闭，close all swap devices
# 修改/etc/fstab，注释掉swap那行，持久化生效
# sudo vim /etc/fstab
```



## 关闭防火墙 iptables 开放路由策略(所有虚拟机)

```sh
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
```



## 为各个服务器节点配置好免密登录(所有虚拟机)

在每个服务分别执行

```sh
ssh-keygen

ssh-copy-id root@192.168.3.90
ssh-copy-id root@192.168.3.91
ssh-copy-id root@192.168.3.92

```



## 为各个服务器节点设置hostname(所有虚拟机)

```sh
hostnamectl set-hostname k8s-1
hostnamectl set-hostname k8s-2
hostnamectl set-hostname k8s-3
```



## 检查并安装必要的模块(所有虚拟机)

创建 `module-check+install.sh` 并执行该文件：

```sh
#!/bin/zsh
for module in br_netfilter ip6_udp_tunnel ip_set ip_set_hash_ip ip_set_hash_net iptable_filter iptable_nat iptable_mangle iptable_raw nf_conntrack_netlink nf_conntrack nf_conntrack_ipv4   nf_defrag_ipv4 nf_nat nf_nat_ipv4 nf_nat_masquerade_ipv4 nfnetlink udp_tunnel veth vxlan x_tables xt_addrtype xt_conntrack xt_comment xt_mark xt_multiport xt_nat xt_recent xt_set  xt_statistic xt_tcpudp; do
    if ! lsmod | grep -q $module; then
        echo "module $module is not present, try to install...";
                modprobe $module
                if [ $? -eq 0 ]; then
                        echo -e "\033[32;1mSuccessfully installed $module!\033[0m"
                else
                        echo -e "\033[31;1mInstall $module failed!!!\033[0m"
                fi
        fi;
done
```



## 网桥设置(所有虚拟机)

```sh
#!/bin/zsh
echo "fix the net.bridge.bridge-nf-call-iptables=1 with fllowing lines"
echo "cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system"
```





## rke部署k8s (master 节点)

### 下载 rke

下载 rke 的可执行文件： [Releases · rancher/rke](https://link.zhihu.com/?target=http%3A//github.com/rancher/rke/releases)

**国内由于各种原因可能会很慢，这里推荐一个网站：[下载](https://d.serctl.com) 直接起飞**

```
mv rke_linux-amd64 /usr/local/bin/rke
```



### 配置 cluster.yml

```yml
# 集群名称
cluster_name: k8s-demo
authentication:
  strategy: x509|webhook
kubernetes_version: v1.19.9-rancher1-1
nodes:
    # address用于从本地SSH连接到服务器节点
  - address: 192.168.3.90
    # internal_address用于节点之间通信
    internal_address: 192.168.3.90
    user: root
    role: [controlplane, worker, etcd]
  - address: 192.168.3.91
    internal_address: 192.168.3.91
    user: root
    role: [controlplane, worker, etcd]
  - address: 192.168.3.92
    internal_address: 192.168.3.92
    user: root
    role: [controlplane, worker, etcd]
    
# rke数据存储位置
prefix_path: /data/rke
services:
  etcd:
    snapshot: false
    creation: 12h
    retention: 72h
    backup_config:
      enabled: true
      interval_hours: 12
      retention: 6
      safe_timestamp: false
  kube-controller:
    extra_args:
      # 以下两个选项用于启用内置signer，用于后续创建管理员流程
      cluster-signing-cert-file: /etc/kubernetes/ssl/kube-ca.pem
      cluster-signing-key-file: /etc/kubernetes/ssl/kube-ca-key.pem
  kube-api:
    service_node_port_range: 30000-39999
# 使用calico网络
network:
  plugin: calico
# 不使用内置Ingress
ingress:
    provider: nginx
```



### 新建k8s集群

```sh
rke up
```

等待命令行提示 `finished building Kubernetes cluster successfully` 代表集群搭建成功。

在目录下会生产 kube_config_cluster.yml、cluster.rkestate文件。

- cluster.yml，kube_config_cluster.yml文件一定要好好保存，后续集群维护都要用到

- - kube_config_cluster.yml这个文件有`kubectl`和`helm`的凭据。
  - rancher-cluster.yml：RKE集群配置文件。
  - kube_config_rancher-cluster.yml：集群的 [Kube config文件](https://link.zhihu.com/?target=https%3A//rancher.com/docs/rke/latest/en/kubeconfig/)，此文件包含完全访问集群的凭据。
  - cluster.rkestate：[Kubernetes集群状态文件](https://link.zhihu.com/?target=https%3A//rancher.com/docs/rke/latest/en/installation/%23kubernetes-cluster-state)，此文件包含完全访问集群的凭据。



## 验证并使用k8s集群

### 安装 `kubectl` 

```sh
apt-get update
apt-get install -y ca-certificates curl software-properties-common apt-transport-https curl
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

tee /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubectl 
```

### 创建 ` ~/.kube` 以及config

```sh
cd 
mkdir .kube
cp kube_config_cluster.yml .kube/config
```

### 使用 `kubectl` 查看集群

```sh
kubectl get nodes # 查看节点信息
```


## rke 升级 k8s 版本

执行一下命令查看当前 rke 版本支持升级哪些 k8s 版本

```sh
rke config --list-version --all
```

找到原来的 `cluster.yml` 文件，替换以下配置信息

```yaml
kubernetes_version: "v1.21.6-rancher1-1"
```

执行升级命令，等待升级完成即可

```sh
rke up --config cluster.yml
```















