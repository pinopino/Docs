ElasticSearch的一些常见操作
=========

一个简单的select *：
GET /megacorp/employee/_search
分别指定了索引，类型，以及select的方式为*

集群的监控统计信息中最重要的一个指标就是`集群健康`：
GET /_cluster/health
status有三个值：green、yellow以及red