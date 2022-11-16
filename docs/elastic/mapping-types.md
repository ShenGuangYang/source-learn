# [字段数据类型](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/mapping-types.html) 

## [text](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/text.html) 

主要用于全文检索领域

```
PUT person-001
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "hobby": {
        "type": "text",
        "analyzer": "standard"
      }
    }
  }
}

POST person-001/_doc
{
  "name": "shen",
  "hobby": "basketball, football, pingpong"
}


POST _analyze
{
  "analyzer": "standard",
  "text": ["I like basetball!!!"]
}
```



## [keywords](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/keyword.html#keyword) 

主要用于固定精准检索领域

关键字族包括以下字段类型：

- [`keyword`](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/keyword.html#keyword-field-type)，用于精确检索
- [`constant_keyword`](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/keyword.html#constant-keyword-field-type) 需要设定默认固定值，默认是填充值，便于查询与统计，避免错误或者性能问题
- [`wildcard`](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/keyword.html#wildcard-field-type) 采用 ngram 模型分词，支持通配模式查询



```
PUT person-001
{
  "mappings": {
    "properties": {
      "name01": {
        "type": "keyword"
      },
      "name02": {
        "type": "constant_keyword",
        "value": "default_value"
      },
      "name03": {
        "type": "wildcard"
      }
    }
  }
}
# 新建失败
PUT person-001/_doc/1
{
  "name01": "shen",
  "name02": "shen",
  "name03": "shen"
}
# 新建成功
PUT person-001/_doc/2
{
  "name01": "shen",
  "name03": "shen"
}

GET person-001/_search
```



## [number](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/number.html#number) 



数值字段类型

| 类型            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| `long`          | 64位整数，范围 $-2^{63}$ -  $2^{63}-1$                       |
| `integer`       | 32位整数，范围 $-2^{32}$ -  `$2^{32}-1$ `                    |
| `short`         | 16位整数，范围  `-32768`  - `32767`                          |
| `byte`          | 8位整数，范围 `-128` - `127`                                 |
| `double`        | 64位双精度                                                   |
| `float`         | 32位单精度                                                   |
| `half_float`    | 16位半精度                                                   |
| `scaled_float`  | 收缩浮点类型，背后基于 long 类型实现 ，需要设置收缩因子，scaling_factor |
| `unsigned_long` | 64位整数，0 - $2^{64}-1$                                     |



## [date](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/date.html) 





















