Elasticsearch中的Mapping和Analysis
=========

ES中存储的文档，数据实际最终是被存储在lucene上的。而lucene仅仅支持键值对方式的扁平化存储，要将这个json文档存储到lucene中我们除了需要将有层级的json对象扁平化为`[字段:值]`方式的键值对，还需要解决下面两个问题：
  1. json对象并没有携带关于字段类型的信息
  2. 如何为string类型的字段建立倒排索引
   
ES中的两个概念映射（mapping）和分析（analysis）就是用来解决上述两个问题的。

通过mapping我们将json字段匹配到lucene中一种确定的数据类型，lucene支持的常见数据类型有：`text`、`keyword`、`date`、`byte`、`short`、`integer`、`long`、`float`、`double`、`boolean`；

还有一些类型是属于mapping的元数据，常见的都有：`_index`、`_type`、`_id`

mapping要么跟着index新建的时候一起指定好，要么就只能做加法修改。核心的原则就是mapping的修改不能影响到之前已经索引好的文档（因为文档是不可变的），那么能做的修改只还剩下：
- 添加一个新的要映射的字段
- 为已有映射字段添加一个新的映射，`multi-fields`
- 将已有映射字段给忽略掉，`ignore_above`

下面是几个例子：
```csharp
// 初始创建索引，没有携带任何mapping信息
PUT twitter 
{}

// 创建mapping，为_doc类型添加了一个email字段
PUT twitter/_mapping/_doc 
{
  "properties": {
    "email": {
      "type": "keyword"
    }
  }
}
```

