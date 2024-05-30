# systemd

## unit

### unit 的含义

Systemd 可以管理所有系统资源。不同的资源统称为 Unit（单位）。

```bash
# 列出正在运行的 Unit
systemctl list-units

# 列出所有Unit，包括没有找到配置文件的或者启动失败的
systemctl list-units --all

# 列出所有没有运行的 Unit
systemctl list-units --all --state=inactive

# 列出所有加载失败的 Unit
systemctl list-units --failed

# 列出所有正在运行的、类型为 service 的 Unit
systemctl list-units --type=service
```

### unit 的状态

```bash
# 显示系统状态
systemctl status

# 显示单个 Unit 的状态
sysystemctl status bluetooth.service

# 显示远程主机的某个 Unit 的状态
systemctl -H root@rhel7.example.com status httpd.service

# 显示某个 Unit 是否正在运行
systemctl is-active application.service

# 显示某个 Unit 是否处于启动失败状态
systemctl is-failed application.service

# 显示某个 Unit 服务是否建立了启动链接
systemctl is-enabled application.service
```

### unit 的管理

```bash
# 立即启动一个服务
sudo systemctl start apache.service

# 立即停止一个服务
sudo systemctl stop apache.service

# 重启一个服务
sudo systemctl restart apache.service

# 杀死一个服务的所有子进程
sudo systemctl kill apache.service

# 重新加载一个服务的配置文件
sudo systemctl reload apache.service

# 重载所有修改过的配置文件
sudo systemctl daemon-reload

# 显示某个 Unit 的所有底层参数
systemctl show httpd.service

# 显示某个 Unit 的指定属性的值
systemctl show -p CPUShares httpd.service

# 设置某个 Unit 的指定属性
sudo systemctl set-property httpd.service CPUShares=500
```

### 依赖关系

Unit 之间存在依赖关系：A 依赖于 B，就意味着 Systemd 在启动 A 的时候，同时会去启动 B。

`systemctl list-dependencies` 命令列出一个 Unit 的所有依赖。

```bash
systemctl list-dependencies nginx.service
```

上面命令的输出结果之中，有些依赖是 Target 类型（详见下文），默认不会展开显示。如果要展开 Target，就需要使用--all参数。

```bash
systemctl list-dependencies --all nginx.service
```

## 在 systemd 上运行 jar

1. 创建 systemd service file

```bash
sudo cat <<EOF> /etc/systemd/system/cassandra-export.service
[Unit]
Description=cassandra_exporter
After=network.target

[Service]
ExecStart=/usr/bin/java -Xms128m -Xmx256m -jar /usr/local/lib/cassandra_exporter-2.3.5.jar /etc/cassandra/cassandra_exporter-config.yml
Type=simple
User=nobody
Group=nogroup
Restart=always
TimeoutStopSec=60

[Install]
WantedBy=multi-user.target

EOF
```

2. 重新加载 systemd

```bash
sudo systemctl daemon-reload
```

3. 启动应用并设置开机启动

```bash
systemctl start cassandra-export.service
systemctl enable cassandra-export.service
```

# journalctl

Systemd 统一管理所有 Unit 的启动日志。带来的好处就是，可以只用journalctl一个命令，查看所有日志（内核日志和应用日志）。日志的配置文件是/etc/systemd/journald.conf。



```bash
# 查看系统本次启动的日志
sudo journalctl -b
sudo journalctl -b -0


# 查看指定时间的日志
sudo journalctl --since="2023-10-30 18:17:16"
sudo journalctl --since "20 min ago"
sudo journalctl --since yesterday
sudo journalctl --since "2024-01-10" --until "2024-01-11 03:00"
sudo journalctl --since 09:00 --until "1 hour ago"

# 显示尾部的最新10行日志
sudo journalctl -n

# 显示尾部指定行数的日志
sudo journalctl -n 20

# 实时滚动显示最新日志
sudo journalctl -f

# 查看指定服务的日志
sudo journalctl /usr/lib/systemd/systemd
# 比如查看docker服务的日志
sudo systemctl status docker

# 查看指定进程的日志
sudo journalctl _PID=1

# 查看某个路径的脚本的日志
sudo journalctl /usr/bin/bash

# 查看指定用户的日志
sudo journalctl _UID=33 --since today

# 查看某个 Unit 的日志
sudo journalctl -u nginx.service
sudo journalctl -u nginx.service --since today

# 实时滚动显示某个 Unit 的最新日志
sudo journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
journalctl -u nginx.service -u php-fpm.service --since today



# 显示日志占据的硬盘空间
sudo journalctl --disk-usage

# 指定日志文件占据的最大空间
sudo journalctl --vacuum-size=1G

# 指定日志文件保存多久
sudo journalctl --vacuum-time=1years
```