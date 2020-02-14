ElasticSearch的一些基础概念-part2
=========
#### 以下描述均基于ES 6.x版本
#### 路由一个文档到一个分片中
当索引一个文档的时候，文档会被存储到一个主分片中。ES如何知道一个文档应该存放到哪个分片中呢？当我们创建文档时，它如何决定这个文档应当被存储在分片1还是分片2中呢？

ES实际上采用下面的公式进行路由：
`shard = hash(routing) % number_of_primary_shards`
`routing`是一个可变值，默认是文档的_id，也可以设置成一个自定义的值。routing通过hash函数生成一个数字，然后这个数字再除以number_of_primary_shards（主分片的数量）后得到余数。这个分布在0到number_of_primary_shards-1之间的余数，就是我们所寻求的文档所在分片的位置。

这里实际上也解释了为什么创建index的时候，主分片的数量会需要固定下来并且不可再改变的原因。

#### 多索引，多类型搜索
ES最原始的搜索支持在所有索引的所有分片上搜索所有类型的文档数据！显然，这样的效率不是最高的，因此也提供各种范围选定以便可以在一个较小的范围集上做搜索查询：
```csharp
// 在所有的索引中搜索所有的类型
/_search

// 在 gb 索引中搜索所有的类型
/gb/_search

// 在 gb 和 us 索引中搜索所有的文档
/gb,us/_search

// 在任何以 g 或者 u 开头的索引中搜索所有的类型
/g*,u*/_search

// 在 gb 索引中搜索 user 类型
/gb/user/_search

// 在 gb 和 us 索引中搜索 user 和 tweet 类型
/gb,us/user,tweet/_search

// 在所有的索引中搜索 user 和 tweet 类型
/_all/user,tweet/_search
```
如果为上述http请求附加上body信息（这被称为查询表达式`Query DSL`），则可以描述更加复杂的[查询需求](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_most_important_queries.html)。当使用DSL进行查询时分两种情况讨论：
- 过滤（`filter`）
即通常所说的精确匹配，不评分查询
- 查询（`query`）
即通常所说的全文检索，评分查询

##### match查询
如果你在一个全文字段上使用match查询，在执行查询前ES将用正确的分析器去分析查询字符串：
```csharp
{ "match": { "tweet": "About Search" }}
```
而如果在一个精确值的字段上使用它，例如数字、日期、布尔或者一个 not_analyzed 字符串字段，那么它将会精确匹配给定的值：
```csharp
{ "match": { "age":    26           }}
{ "match": { "date":   "2014-09-01" }}
{ "match": { "public": true         }}
{ "match": { "tag":    "full_text"  }}
```
当然，ES提供了专门的精确值匹配查询关键字`term`（`terms`）：
```csharp
{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
// terms，如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件：
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```

##### multi_match查询
multi_match查询可以在多个字段上执行相同的match查询：
```csharp
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}
```

##### range查询
range查询找出那些落在指定区间内的数字或者时间：
```csharp
{
    "range": {
        "age": {
            "gte":  20, // gt >; gte >=; lt <; lte <=;
            "lt":   30
        }
    }
}
```

##### exists和missing查询
exists和missing查询被用于查找那些指定字段中有值（exists）或无值 （missing）的文档。这与SQL中的IS NULL和NOT NULL在本质上类似：
```csharp
{
    // 这些查询经常用于某个字段有值和某个字段缺值的情况
    "exists":   {
        "field":    "title"
    }
}
```

##### 组合多查询
通过组合查询来应付复杂的查询需求，组合查询用bool查询来实现，它接受以下参数：
- **must**
文档 ***必须*** 匹配这些条件才能被包含进来
- **must_not**
文档 ***必须不*** 匹配这些条件才能被包含进来
- **should**
如果满足这些语句中的任意语句，将增加_score，否则无任何影响。它们主要用于修正每个文档的相关性得分
- **filter**
必须匹配，但它以不评分、过滤的模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档

一个较为复杂的例子可以是：
```csharp
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "bool": { // bool也可以嵌套
              "must": [
                  { "range": { "date": { "gte": "2014-01-01" }}},
                  { "range": { "price": { "lte": 29.99 }}}
              ],
              "must_not": [
                  { "term": { "category": "ebooks" }}
              ]
          }
        }
    }
}
```

##### 排序与相关性
默认情况下，ES中返回的结果是按照相关性进行[排序](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_Sorting.html)的——最相关的文档排在最前。具体来说ES中相关性得分由一个浮点数进行表示，并在搜索结果中通过_score参数返回，默认排序是_score降序。让我们以一个例子开始：
```csharp
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : {
                "term" : {
                    "user_id" : 1
                }
            }
        }
    }
}
```
在这样一个查询中评分是没有任何意义的，因为我们使用的是filter查询（过滤），这表明我们只希望获取匹配user_id=1的文档，并没有试图确定这些文档的相关性。 实际上文档将按照随机顺序返回，并且每个文档的评为都会为零。

但另外的情况中，按照某些字段进行排序是有意义的：
```csharp
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
}
```
或者：
```csharp
GET /_search
{
    "query" : {
        "bool" : {
            "must":   { "match": { "tweet": "manage text search" }},
            "filter" : { "term" : { "user_id" : 2 }}
        }
    },
    // 排序将首先按照date然后按照文档的得分进行排序，_score为ES保留关键字
    "sort": [ 
        { "date":   { "order": "desc" }},
        { "_score": { "order": "desc" }}
    ]
}
```

##### 字符串字段的排序
在实际需求中可能会遇到这样的情况：我们需要对一个字符串字段做全文搜索，同时我们也希望在返回的结果中按照这个字段的值进行排序。直接上解决办法：
```csharp
"tweet": {// tweet主字段与之前的一样：是一个analyzed全文字段
    "type":     "string",
    "analyzer": "english",
    "fields": {
        "raw": {// 新的tweet.raw子字段是not_analyzed的
            "type":  "string",
            "index": "not_analyzed"
        }
    }
}
```
于是便可以愉快的搜索并排序了：
```csharp
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    },
    "sort": "tweet.raw"
}
```

更加复杂深入的查询参考官方文档[这里](https://www.elastic.co/guide/cn/elasticsearch/guide/current/search-in-depth.html)。

##### 验证查询
随着需求变得复杂很快你构建的查询语句也将会复杂起来。我们需要一种可以方便“调试”查询的的方法，ES为我们提供了两个强有力的工具：
- `validate-query`
比如：
```csharp
GET /gb/tweet/_validate/query
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```
会得到：
```csharp
{
  "valid" : false, // 表明上面的查询是错误的
}
```

- `explain`
我们可以进一步通过explain来理解查询为什么错了：
```csharp
GET /gb/tweet/_validate/query?explain 
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```
会得到：
```csharp
{
  "explanations" : [{
    "index" :   "gb",
    "valid" :   false,
    "error" :   "org.elasticsearch.index.query.QueryParsingException:
                 [gb] No query registered for [tweet]"
  }]
}
```
不难看出上面查询之所以会报错是因为我们将查询类型（match）与字段名称（tweet）搞混了。

另外值得一提的是不光是寻求报错的原因时我们使用explain，正确的查询我们一样可以通过explain来了解学习ES是如何解释执行我们的查询的。

##### 关于分页
最后再说一点关于分页的东西。由于ES天生的分布式特性，分页时需要考虑深度分页的问题。问题本质是很简单的比如类似这样一个查询：
```csharp
GET /_search?size=10&from=0
```
上面的查询会在所有分片上都执行。如果我们主分片的数量是5，并且请求前10条记录，那么实际执行中所有分片都需要产生自己的前10位结果并返回给协调节点，随后协调节点对这50个结果进行排序返回前10个。试想如果分页的单位是1000，这样的结果量级会有多大！

#### ES中的sql
(TODO)

#### ES中数据建模
[数据建模](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relations.html)涉及到把业务需求整理出来的对象映射到ES中。

在关系型数据库的建模中每个实体（或行，在关系世界中）可以被主键唯一标识；实体规范化（范式），唯一实体的数据只存储一次而相关实体只存储它的主键；实体可以进行关联查询，可以跨实体搜索，等等。

而ES因为直接存储的是json文档，所以我们需要不同的建模方式来表达对象间的关系，常用的方法有四种：
- `应用层连接`（Application-side joins）
多次查询最后再应用层面进行拼接
- `数据非规范化`（Data denormalization）
即通常所说的冗余，将冗余信息通过内部对象（inner object）或者嵌套对象（nest object）的方式直接附加在原始对象上面
- `父子关系文档`（Parent/child relationships）
- `嵌套对象`（Nested objects）

这里重点解释下最后一个[nest object](https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-objects.html)的概念。我们知道ES是支持json的所有类型的，因此除了常见的标量值还支持复杂类型，比如null、数组和对象。一个复杂json如：
```csharp
{
    "tweet":            "Elasticsearch is very flexible",
    "user": {
        "id":           "@johnsmith",
        "gender":       "male",
        "age":          26,
        "name": { // inner object
            "full":     "John Smith",
            "first":    "John",
            "last":     "Smith"
        }
    }
}
```
会被ES映射成：
```csharp
{
    "tweet":            [elasticsearch, flexible, very],
    "user.id":          [@johnsmith],
    "user.gender":      [male],
    "user.age":         [26],
    "user.name.full":   [john, smith],
    "user.name.first":  [john],
    "user.name.last":   [smith]
}
```
主要原因还是因为lucene并不理解内部对象，所以ES会自动摊平对象嵌套结构。另外一种常见的复杂json可以是：
```csharp
{
    "followers": [
        { "age": 35, "name": "Mary White"},
        { "age": 26, "name": "Alex Jones"},
        { "age": 19, "name": "Lisa Smith"}
    ]
}
```
同样的会被转化映射成为：
```csharp
{
    "followers.age":    [19, 26, 35],
    "followers.name":   [alex, jones, lisa, smith, mary, white]
}
```
执行查询时根据业务来说可能会遇到比较尴尬的情况。比如我们无法回答一个准确的答案：“是否有一个26岁名字叫Alex Jones的追随者？”。`{age: 35}` 和 `{name: Mary White}`之间的相关性已经丢失了，因为每个多值域只是一包无序的值，而不是有序数组。

为了处理这种情况，ES中引入了nest object的概念。只需要将对象字段类型设置为 nested而不是object后，每一个嵌套对象都会被索引为一个隐藏的独立文档：
```csharp
{ // 嵌套文档1
  "comments.name":    [ john, smith ],
  "comments.comment": [ article, great ],
  "comments.age":     [ 28 ],
  "comments.stars":   [ 4 ],
  "comments.date":    [ 2014-09-01 ]
}
{ // 嵌套文档2
  "comments.name":    [ alice, white ],
  "comments.comment": [ like, more, please, this ],
  "comments.age":     [ 31 ],
  "comments.stars":   [ 5 ],
  "comments.date":    [ 2014-10-22 ]
}
{ // 根文档或者叫父文档
  "title":            [ eggs, nest ],
  "body":             [ making, money, work, your ],
  "tags":             [ cash, shares ]
}
```
很显然在独立索引每一个嵌套对象后，对象中每个字段的相关性得以保留，我们查询时也就可以只返回那些真正符合条件的文档了。


