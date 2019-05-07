ElasticSearch的一些基础概念
=========

- Elasticsearch中的几种名词关系（类比关系型数据库）：
    ```csharp
    Relational DB -> Databases -> Tables -> Rows -> Columns
    Elasticsearch -> Indices -> Types -> Documents -> Fields
    ```

- Elasticsearch集群可以包含多个 ***索引***（`indices`），每一个索引可以包含多个 ***类型***（`types`），每一个类型包含多个 ***文档***（`documents`），然后每个文档包含多个 ***字段***（`fields`）。<br/><br/>

- Elasticsearch中实际存储数据的地方是 ***分片***（`shard`），索引只是一个指向一个或者多个分片的逻辑概念；Elasticsearch中的最小工作单元就是分片，一个分片就是一个lucene实例，并且它本身就是一个完整的搜索引擎；<br/><br/>

- 分片可以是主分片也可以是复制分片（类似于关系型数据库的主备，可以防止数据丢失，另外也能分摊压力）；<br/><br/>

- 主分片的数量在创建之初即定好了不可以修改（实际上，这个数量定义了能存储到索引里数据的最大数量（实际的当然数量取决于你的数据、硬件和应用场景）），复制分片的数量却可以修改；<br/><br/>

- 像MongoDB一样，在Elasticsearch中并不需要显式的创建索引（数据库）和类型（表），直接插入一条新的数据Elasticsearch会自动为你创建好；这默认创建好的也算是一个集群，一个单一节点的集群。我们也可以添加更多的节点进来，只要第二个节点与第一个节点含有相同的cluster.name即可（Elasticsearch内部会通过广播自动发现第一个节点所在的集群）；<br/><br/>

- 
