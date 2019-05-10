ElasticSearch的一些常见操作
=========

### 增删改
创建一个索引：
```csharp
PUT /customer
```

索引一个文档：
```csharp
PUT /customer/user/1
{
   "name": "John Doe" 
}

// 或者，使用ES的自生成ID：
// 注意到http动词已经换成了POST
POST /customer/_doc
{
  "name": "Jane Doe"
}
```

获取刚才创建的文档：
```csharp
GET /customer/user/1
```

删除一个索引：
```csharp
DELETE /customer
```

更新一个文档：
```csharp
// 这里相当于是一个reindex动作，意即在当前id下重新放置一个新文档
PUT /customer/user/1
{
  "name": "Jason"
}

// ES也提供了一个update接口
// "doc"是语法规定，这里update不光修改了名称，还添加了一个字段"age"
POST /customer/user/1/_update
{
  "doc": { "name": "Jason", "age": 20 }
}
```

删除一个文档：
```csharp
DELETE /customer/user/2
```

批量操作的简单例子：
```csharp
// 下面操作新添加了两个文档
POST /customer/user/_bulk
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }

// 下面操作修改了一个文档，并且删除了另外一个
// 注意到delete操作并不需要携带操作体
POST /customer/user/_bulk
{"update":{"_id":"1"}}
{"doc": { "name": "John Doe becomes Jane Doe" } }
{"delete":{"_id":"2"}}
```

### 查询

select *：
```csharp
GET /bank/_search
{
  "query": { "match_all": {} }
}

// 也可以不带json体，跟上面效果一样的
GET /bank/_search

// 不想要*，想要指定返回字段
GET /bank/_search
{
  "query": { "match_all": {} },
  "_source": ["account_number", "balance"]
}
```

想要分页：
```csharp
GET /bank/_search
{
  "query": { "match_all": {} },
  "from": 10, // 从0开始计数
  "size": 10
}

// 为上面的分页结果带上排序
GET /bank/_search
{
  "query": { "match_all": {} },
  "from": 10, 
  "size": 10,
  "sort": { "balance": { "order": "desc" } }
}
```

具体带条件的查询（类似于where）
```csharp
// 注意，下面的查询都不再是match_all了

// where account_number = 20
GET /bank/_search
{
  "query": { "match": { "account_number": 20 } }
}

// where address like '%mill%'
GET /bank/_search
{
  "query": { "match": { "address": "mill" } }
}

// where address like '%mill%' or address like '%lane%'
GET /bank/_search
{
  "query": { "match": { "address": "mill lane" } }
}

// where address like '%mill lane%'
GET /bank/_search
{
  "query": { "match_phrase": { "address": "mill lane" } }
}

// where address like '%mill%' and address like '%lane%'
// must指明了必须要两个match同时满足
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}

// 用should可以替换must，表达的是或者的意思，因此前面的
// { "match": { "address": "mill lane" } }也可以翻写为
GET /bank/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}

// 除了有must、should还有一个must_not，这些谓词还能组合在一起
// 以满足更为复杂的查询
// 注意must和must_not在这里是and关系
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
```

_search还有一些有趣的变体：
```csharp
/_search
在所有索引的所有类型中搜索

/gb/_search
在索引 gb 的所有类型中搜索

/gb,us/_search
在索引 gb 和 us 的所有类型中搜索

/g*,u*/_search
在以 g 或 u 开头的索引的所有类型中搜索

/gb/user/_search
在索引 gb 的类型 user 中搜索

/gb,us/user,tweet/_search
在索引 gb 和 us 的类型为 user 和 tweet 中搜索

/_all/user,tweet/_search
在所有索引的 user 和 tweet 中搜索
```

上面的查询都会计算文档的_score，按照相关程度进行返回。如果我们的查询只是需要精确匹配的动作，那么应当使用filter
```csharp
// 
GET /bank/_search
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
```

### 聚合运算
ES中一个比较有特色的地方在于，聚合运算基于的数据集跟最终的聚合运算结果集可以一并返回，方便。

```csharp
// group by state，limit 10（不指定的话默认就是10）
// size指定为0，意即我们不需要返回聚合运算基于的数据集
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}

// select g, avg(balance) group by state
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}

// 
```


**参考链接**
https://www.elastic.co/guide/en/elasticsearch/reference/6.7/index.html