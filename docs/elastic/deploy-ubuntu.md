# ubuntu 安装简单集群安装

前置准备三台 ubuntu 服务器

## 安装 elastic search

```

wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" > /etc/apt/sources.list.d/elastic-7.x.list
apt-get update && apt-get install -y elasticsearch

```



配置文件地址: `/etc/elasticsearch/elasticsearch.yml` 

```yaml
cluster.name: demo-platform
node.name: node-1  # 此项在各节点上分别配置为node-1/2/3
path.data: /data/elasticsearch # 节点上需要授予权限
path.logs: /var/log/elasticsearch
network.host: 192.168.3.80
discovery.seed_hosts: ["192.168.3.80", "192.168.3.81", "192.168.3.82"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
```



## 安装 kibana

选某一个节点部署

```
apt-get install -y kibana
```

配置文件地址： `/etc/kibana/kibana.yml` 

```yaml
server.host: "192.168.3.80"
server.name: "demo-platform"
elasticsearch.hosts: ["http://192.168.3.80:9200", "http://192.168.3.80:9200", "http://192.168.3.80:9200"]
```



## 错误排查



查看日志

```bash
##按服务查看
journalctl -u kibana.service
## 按进程查看
journalctl _PID=5601

```





