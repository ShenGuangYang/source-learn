# Vagrant 简介

## vagrant基本概念
[vagrant](https://www.vagrantup.com/intro)可方便地管理各种类型的虚拟机，它是vmware/virtualbox/hyperv等虚拟化管理工具的上层集成式管理工具、虚拟机自动化配置工具、虚拟机批量管理工具。

简单来说，我们平时在使用 virtualbox 创建linux 系统，通常都需要自己去下载镜像、安装镜像、配置网络等操作。要是搭建各类集群服务时，重复操作太多，vagrant 可以简化此操作过程。（类似于 docker-compose 管理多个docker 容器）


## 安装 vagrant

[官网下载](https://www.vagrantup.com/downloads) 下载软件傻瓜式安装。

## 设置 vagrant_home
vagrant在执行子命令box add、init、up等命令时，都可能会去下载所需的虚拟机镜像文件，即Box image。

这些镜像文件默认放在`~/.vagrant.d`目录(Linux)或`C:\USERS\<NAME>\.vagrant.d\`目录(Windows)下，这些镜像文件一般至少都几百兆，大的镜像可能几个G，所以放在家目录或C盘会占用大量空间。

要修改vagrant下载时默认的镜像保存位置，需设置环境变量 `VAGRANT_HOME`。

- windows 增加环境变量配置
- mac、linux 执行 `echo 'export VAGRANT_HOME="/data/.vagrant.d"' >>~/.bashrc && exec bash`

## vagrant 常用命令

vagrant的子命令不少，可使用vagrant -h列出vagrant默认支持的子命令，使用vagrant list-commands查看vagrant支持的所有子命令(包括因安装插件而增加的子命令)。

这里只是简单概括常用子命令的功能而不介绍如何使用，后面涉及到的时候自然就会用了。

|    子命令     | 功能说明                                                     |
| :-----------: | :----------------------------------------------------------- |
|      box      | 管理box镜像(box是创建虚拟机的模板)                           |
|     init      | 初始化项目目录，将在当前目录下生成Vagrantfile文件            |
|      up       | 启动虚拟机，第一次执行将创建并初始化并启动虚拟机             |
|    reload     | 重启虚拟机                                                   |
|     halt      | 将虚拟机关机                                                 |
|    destroy    | 删除虚拟机(包括虚拟机文件)                                   |
|    suspend    | 暂停(休眠、挂起)虚拟机                                       |
|    resume     | 恢复已暂停(休眠、挂起)的虚拟机                               |
|    status     | 列出当前目录(Vagrantfile所在目录)下安装的虚拟机列表及它们的状态 |
| global-status | 列出全局已安装虚拟机列表及它们的状态                         |
|      ssh      | 通过ssh连接虚拟机                                            |
|  ssh-config   | 输出ssh连接虚拟机时使用的配置项                              |
|   validate    | 校验Vagrantfile                                              |

