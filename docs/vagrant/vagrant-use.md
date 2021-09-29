# Vagrant 使用


## 使用 vagrant 创建并启动一个 CentOS7

进入本地目录 `cd /data` , 然后执行 `vagrant init`：

```shell
cd /data
vagrant init 
```

执行完 `vagrant init` 后，会在当前目录下生成一个 `Vagrantfile` 文件，这个文件定义了将要创建和启动的虚拟机的配置。

打开 `Vagrantfile` 文件，修改或者添加几项内容：

```properties
config.vm.box = "centos7"
config.vm.box_url = "http://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-2002_01.VirtualBox.box"
```

上面的配置表示创建一个 CentOS7 虚拟机，镜像源从指定的url下载。 [官方镜像仓库地址](https://www.junmajinlong.com/virtual/vagrant/vagrant_create_vms/)

然后执行：

```sh
vagrant up
```

会输出一大堆东西，然后提示成功之后，可以直接使用 ` vagrant init` 进入虚拟机。

这个时候就可以直接使用 `vagrant ssh` 进入虚拟机对虚拟机进行操作了。



> vagrant up 过程中出现错误或者卡住，这是很正常的事情，windows + virtualbox 的诡异问题，如果在hyperv下安装相同的box没问题，几乎可以肯定是wsl2或win10开启了虚拟化(hyperv)导致的问题。
>
>
> https://stackoverflow.com/questions/64120030/hash-sum-mismatch-when-apt-get-update-ubuntu-20-04-vm-with-multipass/64157969#64157969



## 使用 vagrant box 做虚拟机模板



`box` 是一个开箱即用的虚拟机，可直接由 `vagrant` 启动运行。

每一个 `box` 都是由他人打包好的虚拟机，只不过它是特殊格式的文件，且后缀名一般为 `.box` 。我们也可以使用 `vagrant package` 打包自己的虚拟机并分发给别人使用。

安装一个 `box` ，相当于提供了一个 `base image` ，即虚拟机模板。之后就可以基于这个模板去创建新的虚拟机并启动，`vagrant` 将自动从 `box` 导入虚拟机所需数据。

```sh
# 自动从vagrant官方的仓库中搜索centos/7，
# 这种添加方式，在国内速度可能会非常慢
vagrant box add centos/7
vagrant box add centos/7 --provider virtualbox

# URL方式，可自定义添加后的box名称
vagrant box add centos-7 http://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-2004_01.VirtualBox.box

# 添加本地Box文件，可自定义添加后的Box名称
vagrant box add centos_7 V:\vagrant_imgs\CentOS-7-x86_64-Vagrant-2004_01.VirtualBox.box
```

添加 `box` 后，已经添加的 `box` 名称就可以直接作为虚拟机模板。比如：

```sh
vagrant init centos-7
vagrant up
```

另外，对于vagrant官方仓库的所有box都可以直接使用box名称。例如：

```sh
vagrant init generic/centos7
vagrant init generic/ubuntu2004
vagrant init ubuntu/bionic64
```

当没有指定box镜像路径或URL，就像上面示例直接在init上使用类似xxx作为box时，此时执行 `vagrant up`，`vagrant` 将首先从本地box存放目录 `$HOME/.vagrant.d/或$VAGRANT_HOME/.vagrant.d/` 下寻找名为xxx的box镜像，如果找不到，则自动从官方参考上下载对应的box镜像然后初始化启动，这种方式下载box时可能会很慢。

vagrant支持的管理box的子命令：

```sh
vagrant box -h
Usage: vagrant box <subcommand> [<args>]
subcommands:
  add # 安装box
  list # 列出已安装的box。已安装的box存放在$HOME/.vagrant.d/或$VAGRANT_HOME/.vagrant.d/
  outdated # 检查是否有新版本的box
  prune  # 删除已有新版本的box
  remove # 删除指定的box
  repackage # 重新打包指定的box
  update # 更新box到最新版(不会删除并新建box，而是直接在当前box上更新)
```



## vagrant 配置同步文件夹

同步文件夹能够将宿主机电脑上的文件夹同步到虚拟机中。[官方使用说明](https://www.vagrantup.com/docs/synced-folders/basic_usage)

vagrant 加上同步文件夹功能，对于搭建相同的开发环境、测试环境、搭建中间件的集群环境很有帮助。

目录同步的方式非常简单：

```sh
# 将宿主机项目目录挂载到虚拟机的/vagrant目录
config.vm.synced_folder ".", "/vagrant"

# 将宿主机项目目录内的src子目录挂载到虚拟机的/src/website目录
config.vm.synced_folder "src/", "/srv/website"

# 禁用挂载项：不要在vagrant up或reload时自动挂载
config.vm.synced_folder "src/", "/srv/website", disabled: true

# 修改虚拟机上挂载目录的owner和group
# 不指定owner/group时，它们的owner默认是vagrant ssh所使用的用户，
# 即一般情况下是vagrant用户
config.vm.synced_folder "src/", "/srv/website",
  owner: "root", group: "root"
```



## vagrant 配置虚拟机网络

一般来说，如果没有更多的需求，vagrant对虚拟机网络所做的默认配置应该足够了。

vagrant 配置网络有以下几种方式：

- 端口转发
- 专用网络
- 公共网络



### vagrant配置端口转发



配置端口转发的方式很简单：

```ruby
Vagrant.configure("2") do |config|
  # 虚拟机80端口 <=> 宿主机8080端口
  config.vm.network "forwarded_port", guest: 80, host: 8080

  # 端口映射时，指定绑定的IP地址
  config.vm.network "forwarded_port", 
     guest: 111,  guest_ip: "192.168.1.3", 
     host: 10111, host_up: "192.168.1.5"

  # 指定协议
  config.vm.network "forwarded_port", guest: 222, host: 20222, protocol: "tcp"

  # 宿主机上端口冲突时，自动修正为其他可用端口
  config.vm.network "forwarded_port", guest: 333, host: 30333, auto_correct: true

  # 可指定宿主机上端口冲突时，自动修正时允许的端口范围
  config.vm.usable_port_range = 8000..8999
end
```

可以使用 `vagrant port` 命令查看端口映射情况：

```sh
vagrant port test
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    22 (guest) => 2222 (host)
```



### vagrant 配置 virtualbox 的专用网络

vagrant为virtualbox配置的private_network，其本质是将虚拟机加入到了virtualbox的host-only网络内。



#### DHCP

使用专用网络的最简单方法是允许通过 DHCP 分配 IP。

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "private_network", type: "dhcp"
end
```

这将自动从保留的地址空间分配一个 IP 地址。可以通过使用`vagrant ssh` 进入机器并使用适当的命令行工具查看 IP。



#### 静态IP

您还可以为机器指定静态 IP 地址。这使您可以使用静态的已知 IP 访问 Vagrant 托管机器。静态 IP 的 Vagrantfile 如下所示：

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "private_network", ip: "192.168.50.4"
end
```

> 由用户确保静态 IP 不会与同一网络上的任何其他机器发生冲突。



#### 禁用自动配置

如果你想自己手动配置网络接口，你可以通过指定来禁用 Vagrant 的自动配置功能`auto_config`：

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "private_network", ip: "192.168.50.4",
    auto_config: false
end
```



### vagrant 配置 virtualbox 的公共网络



#### DHCP

使用公共网络的最简单方法是允许通过 DHCP 分配 IP。在这种情况下，定义公共网络非常简单：

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "public_network"
end
```

#### 使用 DHCP 分配的默认路由

在某些情况下，需要保持 DHCP 分配的默认路由不变。在这些情况下，可以指定`use_dhcp_assigned_default_route`选项。举个例子：

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "public_network",
    use_dhcp_assigned_default_route: true
end
```



#### 静态IP

根据您的设置，您可能希望手动设置桥接接口的 IP。

```ruby
config.vm.network "public_network", ip: "192.168.0.17"
```



#### 默认网络接口

如果主机上有多个网络接口可用，Vagrant 会要求您选择虚拟机应该桥接到哪个接口。

```ruby
config.vm.network "public_network", bridge: "en1: Wi-Fi (AirPort)"
```

标识所需接口的字符串必须与可用接口的名称完全匹配。如果找不到，Vagrant 会要求您从可用网络接口列表中进行选择。

对于某些提供程序，可以指定要桥接的适配器列表：

```ruby
config.vm.network "public_network", bridge: [
  "en1: Wi-Fi (AirPort)",
  "en6: Broadcom NetXtreme Gigabit Ethernet Controller",
]
```

在此示例中，将使用存在且可以成功桥接的第一个网络适配器。



#### 禁用自动配置

如果你想自己手动配置网络接口，你可以通过指定来禁用自动配置`auto_config`：

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "public_network", auto_config: false
end
```

然后可以使用shell配置器来配置接口的ip：

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "public_network", auto_config: false

  # manual ip
  config.vm.provision "shell",
    run: "always",
    inline: "ifconfig eth1 192.168.0.17 netmask 255.255.255.0 up"

  # manual ipv6
  config.vm.provision "shell",
    run: "always",
    inline: "ifconfig eth1 inet6 add fc00::17/7"
end
```



## vagrant 管理快照

vagrant 支持为虚拟机拍摄快照来保存当前状态，以后可以方便地恢复到指定的快照。



```sh
vagrant snapshot -h
Usage: vagrant snapshot <subcommand> [<args>]

Available subcommands:
     delete      # 删除快照
     list        # 列出已有的快照
     pop         # 恢复到最近一个快照并删除该快照
     push        # 创建一个快照
     restore     # 恢复到指定的快照
     save        # 创建一个快照，并指定快照名称

```



`save/restore` 和 `push/pop` 是类似的，其实后者是前者的一种简单操作。前者要指定名称，后者自动分配名称。但官方手册建议，不要混用它们，`push` 和 `pop` 一起使用，`save` 和 `restore` 一起使用。

例如：

```sh
# 为虚拟机vm1创建一个快照，快照名为vm1_snapshot1
vagrant snapshot save vm1 vm1_snapshot1

# 使用push创建一个快照，名称自动分配
vagrant snapshot push vm1  # 分配的快照名push_1602923748_6765

# 使用pop恢复到最近一个快照，默认会同时删除该快照
# 使用--no-start可指定恢复快照后不启动虚拟机
# 使用--no-delete可指定pop恢复快照时不删除该快照
vagrant snapshot pop vm1
vagrant snapshot pop vm1 --no-start
vagrant snapshot pop vm1 --no-delete

# 使用restore恢复到指定快照，不会删除任何快照
vagrant snapshot restore vm1 vm1_snapshot1

# 恢复快照后，不要启动虚拟机
vagrant snapshot restore vm1 vm1_snapshot1 --no-start

```





## 自己实际使用的 Vagrantfile

我用这个搭建 k8s 的服务集群、es 集群，效率很高。

```ruby
boxes = [
    {
        :name => "k8s-1",
        :eth1 => "192.168.0.151",
        :mem => "3072",
        :cpu => "2"
    },
    {
        :name => "k8s-2",
        :eth1 => "192.168.0.161",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-3",
        :eth1 => "192.168.0.162",
        :mem => "2048",
        :cpu => "2"
    }
]

ENV["LC_ALL"] = "en_US.UTF-8"

Vagrant.configure(2) do |config|

  config.vm.box = "centos/7"
  
   boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.provider "virtualbox" do |v|
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
	        v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
		      v.customize ["modifyvm", :id, "--name", opts[:name]]
        end
        config.vm.network :public_network, ip: opts[:eth1], bridge: "bridge0"
        config.vm.provision "shell", run: "always", inline: <<-SHELL
          echo "shenshenshen"
		  # 集群搭建好执行的脚本
          # yum -y install wget
        SHELL
      end
  end
end
```



## 宿主机使用 ssh 免密进入虚拟机



```sh
# 先进入虚拟机
vagrant ssh centos7 # centos7 是设置的虚拟机名称，可以使用 vagrant status 查看
## 在虚拟机中的操作
# 进入 root 账号
sudo -i
# 编辑 ssh 配置文件
vi /etc/ssh/sshd_config
# 修改一下内容
PasswordAuthentication yes
PubkeyAuthentication yes
PermitRootLogin yes
# 将本机的 ~/.ssh/id_rsa.pub 复制到 虚拟机的 ~/.ssh/authorized_keys
cp ~/{yours}/.ssh/id_rsa.pub > /root/.ssh/authorized_keys
# 重启 ssh
service sshd restart

# 在宿主机就可以使用 ssh 远程到虚拟机了
```









