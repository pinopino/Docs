ElasticSearch的一些基础概念-part2
=========
#### 以下描述均基于ES 6.x版本
#### bulk批量操作
`mget`，`bulk`

#### 路由一个文档到一个分片中
当索引一个文档的时候，文档会被存储到一个主分片中。ES如何知道一个文档应该存放到哪个分片中呢？当我们创建文档时，它如何决定这个文档应当被存储在分片1还是分片2中呢？

ES实际上采用下面的公式进行路由：
`shard = hash(routing) % number_of_primary_shards`
`routing`是一个可变值，默认是文档的_id，也可以设置成一个自定义的值。routing通过hash函数生成一个数字，然后这个数字再除以number_of_primary_shards（主分片的数量）后得到余数。这个分布在0到number_of_primary_shards-1之间的余数，就是我们所寻求的文档所在分片的位置。

这里实际上也解释了为什么创建index的时候，主分片的数量会需要固定下来并且不可再改变的原因。

#### 多索引，多类型搜索
ES最原始的搜索支持在所有索引的所有分片上搜索所有类型的文档数据！显然，这样的效率不是最高的，因此也提供各种范围选定以便可以在一个较小的范围集上做搜索查询：
```csharp
/_search
在所有的索引中搜索所有的类型

/gb/_search
在 gb 索引中搜索所有的类型

/gb,us/_search
在 gb 和 us 索引中搜索所有的文档

/g*,u*/_search
在任何以 g 或者 u 开头的索引中搜索所有的类型

/gb/user/_search
在 gb 索引中搜索 user 类型

/gb,us/user,tweet/_search
在 gb 和 us 索引中搜索 user 和 tweet 类型

/_all/user,tweet/_search
在所有的索引中搜索 user 和 tweet 类型
```

另外，由于ES天生的分布式特性，搜索并进行分页时需要考虑深度分页的问题。问题本质是很简单的，类似这样一个查询：
`GET /_search?size=10&from=0`
会在所有分片上都查询。如果我们主分片的数量是5，并且请求前10条记录。那么，实际执行中所有分片都需要产生自己的前10位结果并返回给协调节点，随后协调节点对这50个结果进行排序返回前10个。试想如果分页的单位是1000，这样的结果量级会有多大！

#### 复杂核心域类型
ES支持json的所有类型，因此除了常见的标量值，还支持复杂类型，比如null值，数组，和对象。

一个复杂json如：
```csharp
{
    "tweet":            "Elasticsearch is very flexible",
    "user": {
        "id":           "@johnsmith",
        "gender":       "male",
        "age":          26,
        "name": {
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
主要原因还是因为Lucene并不理解内部对象，所以ES会自动摊平对象嵌套结构。

另外一种常见的复杂json可以是：
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
执行查询，根据业务来说可能会遇到比较尴尬的情况。比如我们无法回答一个准确的答案：“是否有一个26岁名字叫Alex Jones的追随者？”。`{age: 35}` 和 `{name: Mary White}`之间的相关性已经丢失了，因为每个多值域只是一包无序的值，而不是有序数组。我们需要另外一个概念来解决这个问题，嵌套对象。

#### 配置分析器
以`standard`分析器为例，它包括了：
- standard分词器，通过单词边界分割输入的文本
- standard语汇单元过滤器，目的是整理分词器触发的语汇单元（但是目前什么都没做）
- lowercase语汇单元过滤器，转换所有的语汇单元为小写
- stop语汇单元过滤器，删除停用词—​对搜索相关性影响不大的常用词，如a，the，and，is等等
  
默认情况下，停用词过滤器是被禁用的。如需启用它，可以通过创建一个基于 standard分析器的自定义分析器并设置stopwords参数。可以给分析器提供一个停用词列表，或者告知使用一个基于特定语言的预定义停用词列表：
```csharp
PUT /spanish_docs
{
    "settings": {
        "analysis": {
            "analyzer": {
                "es_std": {
                    "type":      "standard",
                    "stopwords": "_spanish_"
                }
            }
        }
    }
}
```
注意到es_std分析器不是全局的—​它仅仅存在于我们定义的spanish_docs索引中。一个更复杂的自定义组合分析器的例子可以参考[这里](https://www.elastic.co/guide/cn/elasticsearch/guide/current/custom-analyzers.html)。

#### 动态映射
我们知道ES在遇到文档中以前未遇到的字段时，它会用dynamic mapping来确定字段的数据类型并自动把新的字段添加到类型映射。

这个事实在大多数情况下不需要人工的介入也没有什么思考负担。但是实际中如果一个已经建模好的业务模型随着需求的变更发生了改变，或者一次或者多次。这个时候有可能dynamic mapping的默认行为是不合适的，这就要求有人工介入进来。好在这个默认行为是可配置的：
- `true`（默认值）
  动态添加新的字段
- `false`
  忽略新字段
- `strict`
  如果遇到新字段则抛出异常

```csharp
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic":      "strict", 
            "properties": {
                "title":  { "type": "string"},
                "stash":  {
                    "type":     "object",
                    "dynamic":  true 
                }
            }
        }
    }
}
```
如上，针对my_type类型来说，我们关闭了其动态mapping的功能，但是在其上一个名叫stash，类型为object的字段上我们又开启了动态mapping。这样一来产生的结果便是，如果遇到新字段，对象my_type就会抛出异常；而内部对象stash遇到新字段就会动态创建新字段。

当然，当开启了动态映射之后其本身的默认行为可能不合适，这个在ES中也是可以[配置](https://www.elastic.co/guide/cn/elasticsearch/guide/current/custom-dynamic-mapping.html#dynamic-templates)的。使用`dynamic_templates`，我们可以完全控制新检测生成字段的映射，甚至可以通过字段名称或数据类型来应用不同的映射。

#### 重新索引数据
尽管可以增加新的类型到索引中，或者增加新的字段到类型中，但是不能添加新的分析器或者对现有的字段做改动。如果你那么做的话，结果就是那些已经被索引的数据就不正确，搜索也不能正常工作。

对现有数据的这类改变最简单的办法就是重新索引：用新的设置创建新的索引并把文档从旧的索引复制到新的索引。`_source`字段现在的优点体现出来了，它使得我们在ES中已经有整个文档，这样就不必从源数据中重建索引。

[迁移重建索引](https://www.elastic.co/guide/cn/elasticsearch/guide/current/reindex.html)基本思路就是用`scroll`从旧的索引检索批量文档，然后用`bulk`API把文档推送到新的索引中。

有一个不大不小的问题需要解决，既然是原有索引与新索引同时存在的方式重建索引，那么必然新索引的名字不能与既有的相同。我们得提供一个新索引名称或者使用`索引别名`来解决这个问题。

使用新索引名称意味着上层应用必须更新引用的索引名称。而使用别名则可以绕过这一尴尬，另外别名还可以提供的好处有：
- 在运行的集群中可以无缝的从一个索引切换到另一个索引
- 给多个索引分组 (例如，last_three_months)
- 给索引的一个子集创建`视图`

下面单独解释下怎样使用别名在零停机下从旧索引切换到新索引：
首先，创建索引`my_index_v1`，然后将别名`my_index`指向它：
```csharp
PUT /my_index_v1
PUT /my_index_v1/_alias/my_index
// 额外的，查看my_index指向了哪些真实索引
GET /*/_alias/my_index
// 又或者指定的真实索引使用了哪些别名
GET /my_index_v1/_alias/*
```

然后我们假设需要修改mapping，因此导致需要创建新的索引：
```csharp
PUT /my_index_v2
{
    "mappings": {
        ...new settings...
    }
}
```

接着使用上面提到的迁移重建方法进行数据迁移，一旦我们确定文档已经被正确地重索引了，我们就将别名指向新的索引。一个别名可以指向多个索引，所以我们在添加别名到新索引的同时必须从旧的索引中删除它。这个操作需要原子化，这意味着我们需要使用`_aliases`操作：
```csharp
POST /_aliases
{
    "actions": [
        { "remove": { "index": "my_index_v1", "alias": "my_index" }},
        { "add":    { "index": "my_index_v2", "alias": "my_index" }}
    ]
}
```
至此，我们的应用应该已经在零停机的情况下从旧索引迁移到新索引了。

#### 集群扩容
初始状态下我们可能会这样子搞：
```csharp
PUT /my_index
{
  "settings": {
    "number_of_shards":   1, 
    "number_of_replicas": 0
  }
}
```
但是这个简单来说是一个无扩容因子的集群。因此，当访问压力上去的时候我们试图通过增添一台新的机器是不会有任何效果的。此种情况中没有任何的分片数据需要转移重新负载。我们也不可能调整主分片的数目，因为它本身就是确定之后不可变的。此时唯一能做的是将数据重新索引至一个拥有更多分片的一个更大的索引，但这样做可能意味着线上服务的停机。

基于上面的尴尬情况，很可能又会导致我们走向另一个极端即，我不知道这个索引将来会变得多大，并且过后我也不能更改索引的大小，所以为了保险起见，还是给它设为 1000 个分片吧。

这太极端了，所以正确的做法应该是做容量规划评估：
- 基于你准备用于生产环境的硬件创建一个拥有单个节点的集群
- 创建一个和你准备用于生产环境相同配置和分析器的索引，但让它只有一个主分片无副本分片
- 索引实际的文档（或者尽可能接近实际）
- 运行实际的查询和聚合（或者尽可能接近实际）

基本来说，我们需要复制真实环境的使用方式并将它们全部压缩到单个分片上直到它“挂掉”。实际上挂掉的定义也取决于你：一些用户需要所有响应在50毫秒内返回；另一些则乐于等上5秒钟。

一旦定义好了单个分片的容量，很容易就可以推算出整个索引的分片数。用你需要索引的数据总数加上一部分预期的增长，除以单个分片的容量，结果就是最终需要的主分片个数。

如果确实需要扩容并且不希望停止线上正在运行的服务我们可以怎么做呢？相较于停服然后数据整体迁移到高配索引节点上，我们还有一个可供参考的有用思路：
- 创建一个新的索引来存储新的数据
- 同时搜索两个索引来获取新数据和旧数据

索引别名在这种情况下又可以发挥作用了：
```csharp
PUT /tweets_1/_alias/tweets_search 
PUT /tweets_1/_alias/tweets_index 
```
两个指向同一个索引的别名，一个用于搜索另一个用于索引数据。新文档应当索引至tweets_index；同时，搜索请求应当对别名tweets_search发出。

当进行扩容的时候，我们增加一个名为tweets_2的索引，并且像这样更新别名：
```csharp
POST /_aliases
{
  "actions": [
    { "add":    { "index": "tweets_2", "alias": "tweets_search" }}, 
    { "remove": { "index": "tweets_1", "alias": "tweets_index"  }}, 
    { "add":    { "index": "tweets_2", "alias": "tweets_index"  }}  
  ]
}
```
如此，新索引用来接受读写请求，旧有的索引仅仅只读并不会再写。一个搜索请求可以以多个索引为目标，所以将搜索别名指向tweets_1以及tweets_2是完全有效的。然而，索引写入请求只能以单个索引为目标。因此，我们必须将索引写入的别名只指向新的索引。当然这本身也符合当前的情况，因为确实是因为旧索引容量已达上限才这样搞的。


