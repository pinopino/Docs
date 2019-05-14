Elasticsearch中的Mapping和Analysis
=========

## Analysis

关于analysis你需要意识到一点，索引一个文档以及查询的时候其实都会需要用到分词器的，这个时候通常二者所用的分词器应该要一致才好。

索引文档的时候，如果我们没有指定任何分词器，那么默认ES就会使用`standard analyzer`；而查询时分词器的确定ES有一个优先级：
- An analyzer specified in the query itself
- The search_analyzer mapping parameter
- The analyzer mapping parameter
- An analyzer in the index settings called default_search
- An analyzer in the index settings called default
- The standard analyzer

能够看到，最终如果什么都没有指定的话默认值也会是`standard analyzer`；而另外，在每一次的查询中直接指定analyzer将会极大的方便我们测试查询：
```csharp
// 不需要去指定索引，因为这只是一个独立的测试语句，与具体的索引无关
// 可以每次都换用不同的analyzer测试效果
POST /_analyze
{
  "analyzer": "whitespace",
  "text":     "The quick brown fox."
}
```

我们有必要再来看看`analyzer`，在ES中它的定义很简单，就是由三部分打包组成的一个整体：
- 字符过滤器（character filters）
比如，能够去除HTML标记又或者转换 "&" 为 "and"；另外，多个字符过滤器可以串联起来按顺序执行；
- 分词器（tokenizers）
经过分词器的分词我们才能得到一个个的token，以供下一步标记过滤器使用；不像字符过滤器，一个analyzer只能含有一个分词器
- 标记过滤器（token filters）
比如，`lowercase` token filter能够将所有token全部转换为小写；又或者，一个`stop` token filter能够删掉token所有常见的停词；标记过滤器也能够多个串联起来按顺序执行；

这意味着关于`analyzer`我们可以任意替换其打包中的三个组件来创建属于我们自己的`custom analyzer`：
```csharp
// 注意因为是自定义的，所以这里type属性值一定是"custom"
// 注意analyzer的设置是放在索引的"settings"节中的，并没有在"mapping"中
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type":      "custom", 
          "tokenizer": "standard",
          "char_filter": [
            "html_strip"
          ],
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  }
}

POST my_index/_analyze
{
  "analyzer": "my_custom_analyzer",
  "text": "Is this <b>déjà vu</b>?"
}

// 我们甚至还可以对这三个组件分别进行更详细的设置
// 事实上这也提醒我们，在使用一些第三方analyzer时尽量多去看看有些什么
// 设置可以定制的
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom",
          "char_filter": [
            "emoticons" 
          ],
          "tokenizer": "punctuation", 
          "filter": [
            "lowercase",
            "english_stop" 
          ]
        }
      },
      "tokenizer": {
        "punctuation": { 
          "type": "pattern",
          "pattern": "[ .,!?]"
        }
      },
      "char_filter": {
        "emoticons": { 
          "type": "mapping",
          "mappings": [
            ":) => _happy_",
            ":( => _sad_"
          ]
        }
      },
      "filter": {
        "english_stop": { 
          "type": "stop",
          "stopwords": "_english_"
        }
      }
    }
  }
}

POST my_index/_analyze
{
  "analyzer": "my_custom_analyzer",
  "text":     "I'm a :) person, and you?"
}
```
如果你觉得ES自带的analyzer配置配置也能够用，上面这些有点兴师动众了，那么可以像下面这样：
```csharp
// 我们声明了一个名为"std_english"的analyzer，但是它的类型并不是custom，
// 而是这里的"standard"，这意味着我们仅仅是基于standard做的一些自定义而已
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "std_english": { 
          "type":      "standard",
          "stopwords": "_english_" // "stopwords"是standard analyzer
          // 特有的一个配置属性，所以我们这里才能使用它进行一些定制；
          // 这也提醒我们，如果使用其它第三方的analyzer，多去关注下它们提
          // 供了那些可定制属性
        }
      }
    }
  },
  "mappings": {
    "_doc": {
      "properties": {
        "my_text": {
          "type":     "text",
          "analyzer": "standard", 
          "fields": {
            "english": {
              "type":     "text",
              "analyzer": "std_english" 
            }
          }
        }
      }
    }
  }
}

POST my_index/_analyze
{
  "field": "my_text", 
  "text": "The old brown cow"
}

POST my_index/_analyze
{
  "field": "my_text.english", 
  "text": "The old brown cow"
}
```
上面这段代码也演示了如何为`multi-fields` 指定不同的analyzer，值得一看。


