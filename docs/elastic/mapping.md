# 字段映射



Mapping，是设置索引字段数据类型，限制索引数据模型字段内容行为。



## [动态映射](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/dynamic-field-mapping.html) 

创建索引时未指定字段数据类型时，ES 会动态映射对应的字段数据类型。



## [指定设置映射](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/explicit-mapping.html#explicit-mapping)



```
## 创建索引字段
PUT person-001
{
  "mappings": {
    "properties": {
      "age": {
        "type": "integer"
      },
      "name": {
        "type": "text"
      },
      "email": {
        "type": "keyword"
      }
    }
  }
}

## 添加索引字段
PUT person-001/_mapping
{
  "properties": {
      "age": {
        "type": "integer"
      },
      "name": {
        "type": "text"
      },
      "email": {
        "type": "keyword"
      },
      "gender": {
        "type": "keyword"
      }
    }
}

## 查看索引字段映射
GET person-001/_mapping
## 指定摸一个字段查看映射
GET person-001/_mapping/field/age

```



## 设置 mapping 参数



### [_source field](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/mapping-source-field.html)

在执行 [get](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/docs-get.html#get-source-filtering)  和 [search](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/search-search.html) 请求，在返回 JSON 体里面会含有 `_source` 内容，该字段本身没有被索引（不能搜索），在存储上会有一定的开销。

#### 禁用 _source

```
# 禁用 _source
PUT person-001
{
  "mappings": {
    "_source": {
      "enabled": false
    }, 
    "properties": {
      "name": {
        "type": "text"
      },
      "age": {
        "type": "integer"
      }
    }
  }
}
# 添加数据
POST person-001/_doc
{
  "name": "shen",
  "age": 20
}
# 查询时不返回 _source 内容
GET person-001/_search
# 查询 _source 内容报错
GET person-001/_source/<id>
```



#### 启用 _source

```
# 启用 _source
PUT person-001
{
  "mappings": {
    "_source": {
      "enabled": true
    }, 
    "properties": {
      "name": {
        "type": "text"
      },
      "age": {
        "type": "integer"
      }
    }
  }
}
# 添加数据
POST person-001/_doc
{
  "name": "shen",
  "age": 20
}
# 查询时不返回 _source 内容
GET person-001/_search
# 查询 _source 内容报错
GET person-001/_source/<id>
```



#### 包含、排除 _source

```
PUT person-001
{
  "mappings": {
    "_source": {
      "includes": [
        "*.count",
        "meta.*"
      ],
      "excludes": [
        "meta.description",
        "meta.other.*"
      ]
    }
  }
}

PUT person-001/_doc/1
{
  "requests": {
    "count": 10,
    "foo": "bar" 
  },
  "meta": {
    "name": "Some metric",
    "description": "Some metric description", 
    "other": {
      "foo": "one", 
      "baz": "two" 
    }
  }
}

GET person-001/_search
{
  "query": {
    "match": {
      "meta.other.foo": "one" 
    }
  }
}
```



### [dynamic](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/dynamic.html#dynamic)

动态扩展字段：当数据中包含新字段时，设置该字段会有不同的处理方式。



#### dynamic 参数



| 参数      | 说明                                   |
| --------- | -------------------------------------- |
| `true`    | 默认，容许索引字段动态扩展             |
| `runtime` | 测试阶段，暂不使用                     |
| `false`   | 新字段仅仅存储原始数据，不能被搜索聚合 |
| `strict`  | 禁止扩展新字段                         |



#### dynamic 使用案例



1. false

```
PUT person-001
{
  "mappings": {
    "dynamic": "false",
    "properties": {
      "name": {
        "type": "text"
      }
    }
  }
}

POST person-001/_doc
{
  "name": "shen",
  "age": 20
}
```



2. strict 模式

```
PUT person-001
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name": {
        "type": "text"
      }
    }
  }
}
## 执行以下内容会报错
POST person-001/_doc
{
  "name": "shen",
  "age": 20
}
```



3. true

```
PUT person-001
{
  "mappings": {
    "dynamic": "true",
    "properties": {
      "name": {
        "type": "text"
      }
    }
  }
}

POST person-001/_doc
{
  "name": "shen",
  "age": 20
}
```

















































