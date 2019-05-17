Elasticsearch使用Tips整理
=========

#### 加快导入数据的速度
通常我们会在搭建好ES集群之后导入既有的数据，这个过程根据数据量的大小耗时可长可短，下面是一些可以加快该进程的办法：
- 使用批处理请求
  可能你会想要找到一个合适的bulk值，这可以通过多次迭代测试来实现；比如起始设定一次索引100份文档，之后是200再来是400。总会有一个值从那里开始ES的索引速度会开始降低，那么那个值对于你来说就是一个合适的bulk最大值
- 使用多线程并行插入
  多个线程同时往一个node上插入，还是多个线程同时往多个node上插入，应该都是可以的（？）
  注意在过程中catch住`TOO_MANY_REQUESTS`（429）响应码，对应在client这边Java是以`EsRejectedExecutionException`异常的形式抛出的（C#应该类似）。这个值就是ES在告诉你它现在有点跟不上你的节奏了，所以我们要做的就是接住它并且做延时重插的处理
  同样的，我们也可以通过渐进式的测试来得出线程的合适的数量
- 增加`index.refresh_interval`值（默认值为1），将它设置到一个更大的值比如30s；或者你要导入的数据量实在很大，那么可以考虑直接设置为-1禁用它同时设置`index.number_of_replicas`为0。数据导入完毕之后记得为这些属性重新设置回一个适当值
- 使用自动生成的Id
  ES在插入的时候可以为每个文档生成一个自动Id。如果你想要使用自己的Id，会导致一个问题，即ES需要确认之前插入的没有重复Id（within the same shard）
- 禁止swapping
  基于如何使用ES有两种方式可选
    - 如果服务器的全部资源都用于ES的话 
        ```csharp
        sudo swapoff -a // 临时禁制内存交换
        ``` 
        或者修改/etc/fstab文件，注释掉所有关于swap的项
    - 如果服务器还需要运行其它服务的话
        ```csharp
        // 设置config/elasticsearch.yml
        bootstrap.memory_lock: true
        ``` 

链接：
https://www.elastic.co/guide/en/elasticsearch/reference/6.7/tune-for-indexing-speed.html#_disable_swapping

#### decimal数据类型的处理
从关系型数据库中导入的数据如果含有decimal类型，在ES这边是没有对应类型的，通常情况下推荐使用ES中的`scaled_float`映射类型。该类型原理很简单，[文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/number.html)上是这样描述的：
> A finite floating point number that is backed by a long, scaled by a fixed double scaling factor. 

用一个例子来理解会更容易一些。比如，我们想要存储的值是2.34，如果我们将其映射到scaled_float并且指定scaling_factor为10。那么在ES内部会将其存储为23，并且在所有其它需要用到该字段的地方都会表示成分数的形式，即23/10。我遇到的一个实际的问题在[这里](https://github.com/elastic/elasticsearch/issues/42136)，小心数值大小溢出，虽然实在几率很小了。

额外的，有一个小技巧：如果我们就是想要看看经过放大后被索引进去实际存储下来的值应该怎么做呢？此时_source字段是不起作用的，可以这样：
```csharp
// link：https://stackoverflow.com/questions/55028058/why-is-scaled-float-in-elasticsearch-not-rounding-decimal-places
GET my_index/_search
{
    "size": 0,
    "aggs": {
        "avg": {
            "avg": {
                "field": "price"
            }
        }
    }
}
```
关系型数据库中存储的这些小数类型可以再简单说一下：
- 浮点和定点是两个相对的概念
- 标准定义的浮点数或者定点数本身就不能精确表达某些数值，这不叫精度损失；精度损失是指互相转换的时候数的值发生了改变。比如0.3，二进制无论用什么编码方式都没法精确表达，所以当0.3在内部用浮点编码并使用一个近似值0.30000000000000004来表达的时候，如果我们存储到数据库中类型为float(5,4)的列就会发生截断，这才是精度损失
- 上面的截断动作，在mysql叫截断，会判断截断后一位的值进行rounding（舍入）；而在sql server中则是rouding，会有舍入的动作
- sql server中没有double，只有float。float可以指定(n)，n可取24（4字节保存）或者53（8字节保存），后者即为double；如果选择这些浮点类型直接存储整个0.30000000000000004，也算是"精确"了，没有精度损失


链接：
https://stackoverflow.com/questions/1209181/what-represents-a-double-in-sql-server
https://dev.mysql.com/doc/refman/8.0/en/fixed-point-types.html
https://docs.microsoft.com/en-us/sql/t-sql/data-types/decimal-and-numeric-transact-sql?view=sql-server-2017