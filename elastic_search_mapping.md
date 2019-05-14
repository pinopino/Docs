Elasticsearch中的Mapping和Analysis
=========

## Mapping

ES中存储的文档，数据实际最终是被存储在lucene上的。而lucene仅仅支持键值对方式的扁平化存储，要将这个json文档存储到lucene中我们除了需要将有层级的json对象扁平化为`[字段:值]`方式的键值对，还需要解决下面两个问题：
  1. json对象并没有携带关于字段类型的信息
  2. 如何为string类型的字段建立倒排索引
   
ES中的两个概念映射（mapping）和分析（analysis）就是用来解决上述两个问题的。

通过mapping我们将json字段匹配到lucene中一种确定的数据类型，lucene支持的常见数据类型有：`text`、`keyword`、`date`、`byte`、`short`、`integer`、`long`、`float`、`double`、`boolean`；

还有一些类型是属于mapping的元数据，常见的都有：
- `_index`、`_type`、`_id`，这三个更像是只读属性，在文档返回的时候会一并带回给我们
- `_source`，包含了整个完整的json字符串，它不会被索引（所以也就不可以被搜索到）
- `_all`，就是把文档所有字段全部一股脑塞到一个字段中，ES会为该字段建立倒排索引，因此它是可搜索的；但是该字段本身并不会被存储下来，它仅仅是一个计算值，ES计算出它来，然后建立倒排索引，仅此而已
- `_all`字段最大的用处就是可以让你在对文档类型不了解的情况下，也能够检索文档：
  ```csharp
  // _all默认在ES中是关闭的
  // 在6.0+版本的ES中这个选项已经没法打开了，下面的请求实际上会报错
  PUT /my_index
  {
    "mapping": {
      "user": {
        "_all": {
          "enabled": true   
        }
      }
    }
  }

  PUT /my_index/user/1      
  {
    "first_name":    "John",
    "last_name":     "Smith",
    "date_of_birth": "1970-10-24"
  }

  // 你不用知道user类型有那些字段，直接用_all去检索
  GET /my_index/_search
  {
    "query": {
      "match": {
        "_all": "john smith 1970"
      }
    }
  }
  ```
- 还有一种类似_all的衍生用法，叫自定义_all，是用mapping的`copy_to`参数来实现的。不妨来看个例子：
  ```csharp
  PUT myindex
  {
    "mappings": {
      "mytype": {
        "properties": {
          "first_name": {
            "type":    "text",
            "copy_to": "full_name" 
          },
          "last_name": {
            "type":    "text",
            "copy_to": "full_name" 
          },
          "full_name": {
            "type":    "text"
          }
        }
      }
    }
  }

  PUT myindex/mytype/1
  {
    "first_name": "John",
    "last_name": "Smith"
  }

  GET myindex/_search
  {
    "query": {
      "match": {
        "full_name": "John Smith"
      }
    }
  }
  ```
  例子想表达的意思还是很清晰的。

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

为同一个字段指定不同方式的mapping是非常有用的。比如，你可以将一个string字段映射到text上去，以便做全文搜索；也可以将它映射到keyword，以方便排序或者是聚合查询。在ES中这是通过mapping的`fields`参数来实现的，我们来看一个具体的例子：
```csharp
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "city": {
          "type": "text",
          "fields": {
            "raw": {
              "type":  "keyword"
            }
          }
        }
      }
    }
  }
}

PUT my_index/_doc/1
{
  "city": "New York"
}

PUT my_index/_doc/2
{
  "city": "York"
}

GET my_index/_search
{
  "query": {
    "match": {
      "city": "york" 
    }
  },
  // city.raw现在是keyword类型，正适合用来做排序或聚合
  "sort": {
    "city.raw": "asc"
  },
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.raw" 
      }
    }
  }
}
```
在这段例子中，*city* 不光是被映射成了可搜索的`text`类型，而且通过`field`又映射出来一个类型为`keyword`的*city.raw* 字段！


