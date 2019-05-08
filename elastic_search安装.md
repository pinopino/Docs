ElasticSearch的安装
=========

### window下的安装
在windows下快速的安装，进行快速的验证，上生产环境再考虑切换到linux不失为一个好办法。所以windows下的安装还是有必要了解下，帮助我们能快速开始。

我们选择[zip安装包](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.7.2.zip)进行绿色安装。
> 需要说明的是，官网最新的版本是7.x。而我们之所以使用6.x的版本完全是因为我们要使用的.net client的版本目前只支持到6.x的es；

一个很重要的前提，因为ElasticSearch是用java开发的，所以我们需要事先准备好java的运行环境。好在zip安装包中已经自带了一个完整的java sdk

windows下注意设置好环境变量JAVA_HOME，使其指向es所使用的jdk目录

//////////
关于jdk的选择
https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html
能够看出来

在生产环境中还是建议将conf，data以及logs文件夹与es本身所在的目录分离开来。主要是为了防止升级可能带来的文件删除等误操作
