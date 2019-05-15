Elasticsearch使用Tips整理
=========

#### 加快导入数据的速度
通常我们会在搭建好ES集群之后导入既有的数据，这个过程根据数据量的大小耗时可长可短，下面是一些可以加快该进程的办法：
- 使用批处理请求
  可能你会想要找到一个合适的bulk值，这可以通过多次迭代测试来实现；比如其实设定一次索引100份文档，之后是200再来是400。总会有一个值从那里开始ES的索引速度会开始降低，那么那个值对于你来说就是一个合适的bulk最大值
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

链接：https://www.elastic.co/guide/en/elasticsearch/reference/6.7/tune-for-indexing-speed.html#_disable_swapping
