
# 索引创建

## 创建索引

ES 创建的索引有的场景需要提前创建好，有的场景可以动态创建；
每个索引可以指定一个别名，便于索引重建切换；
索引在创建时需要指定一些参数，有的是固定设置，后期不可以修改，有的是动态设置，后期根据需要进行修改。


### 动态创建
```
POST person-001/_doc
{
    "name": "shen",
    "gender": "man"
}
GET person-001/_search
```

### 静态创建

静态创建指定副本数、分片书、刷新时间等

```
PUT person-001
{
    "settings": {
    "number_of_shards": 2, 
    "number_of_replicas": 2,
    "refresh_interval": "1s"
    }
}
```

### [滚动创建](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/indices-rollover-index.html#indices-rollover-index)

- rollover 是基于索引别名实现，需要优先创建索引并设置别名，
- 完全自动化，需要绑定到 ILM 机制
- 滚动创建索引，通过别名动态的代理索引，执行下一个新的索引机制


#### 创建步骤
1. 创建索引并绑定别名

```
PUT person-rollover-index-000001
{
  "aliases": {
    "person-rollover-index": {}
  }
}
```

2. 设定索引滚动策略

```
POST person-rollover-index/_rollover
{
  "conditions": {
    "max_age": "1d",
    "max_docs": 2,
    "max_size": "1gb"
  }
}
```

#### 触发策略条件

```
POST person-rollover-index/_doc?refresh
{
  "name": "shen"
}
```

> 注意：触发不是实时生效的

#### 手动触发滚动索引

```
## 默认创建下一个索引
POST person-rollover-index/_rollover
## 指定创建下一个索引
POST person-rollover-index/_rollover/person-rollover-index-000010
```

#### 滚动索引的命名规范
- 字母必须小写
- 不能带有特殊符号等
- 不能超过255个字符
- 建议：字母+6位数字，如 xxx-000001

#### 滚动索引触发条件
- 触发条件是非精准型，具有一定延迟性
- max_doc: 索引最大文档数
- max_age: 索引创建时间间隔
- max_size: 索引最大磁盘空间占用





## 绑定别名

所有索引都默认有别名，等同于索引名称，也可以创建与索引名称不一样的索引别名。

```
## 创建索引
PUT person-001
{}

## 设置别名
POST person-001/_alias/person_alias

## 静态创建索引时也可以用如下方式设置别名
PUT person-001
{
  "aliases": {
    "person_alias": {}
  }
}

## 根据别名获取数据
GET person_alias
```



## 索引创建必备设置

- 索引分片数量: 默认1，创建后不可以修改
- 索引副本数量: 默认1，可以动态修改
- 索引刷新间隔: 默认1s，可以动态修改
- 索引别名名称：默认索引名称，可以动态修改



```
## 添加索引
PUT person-index-001
{
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 1,
    "refresh_interval": "1s"
  }, 
  "aliases": {
    "person-index-alias":{}
  }
}

## 更新索引配置
PUT person-index-001/_settings
{
  "settings": {
    "number_of_replicas": 2,
    "refresh_interval": "2s"
  }
}
```



## [集群自动创建禁用机制](https://www.elastic.co/guide/en/elasticsearch/reference/8.5/docs-index_.html#index-creation)



ES 中，动态创建是可以禁止的，某些业务场景需要禁止动态创建索引。



```
# 模式匹配创建索引
PUT _cluster/settings
{
  "persistent": {
    "action.auto_create_index": "+person*,-study*" 
  }
}

POST person-001/_doc
{
    "name": "shen",
    "gender": "man"
}

POST study-001/_doc
{
 "name": "shen",
  "gender": "man"
}
## 禁用自动索引创建
PUT _cluster/settings
{
  "persistent": {
    "action.auto_create_index": "false" 
  }
}
## 允许自动索引创建
PUT _cluster/settings
{
  "persistent": {
    "action.auto_create_index": "true" 
  }
}

```

