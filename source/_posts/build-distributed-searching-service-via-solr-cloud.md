---
title: [solr cloud搭建分布式搜索服务]
date: 2016-06-15 10:51:34
tags: [solr, java]
---

# 前言

搜索服务也基本算是现在互联网产品的标配服务了。目前主流的搜索引擎主要包括[Elasticsearch](https://www.elastic.co/)和[solr](http://lucene.apache.org/solr/)，不过本文并没有关于搜索引擎比较和选择的讨论，主要是讲讲solr cloud使用的一些基本知识。

> solr cloud是solr的分布式模式，用于构建一组solr集群服务器来实现容错和高可用性，提供分布式的索引和搜索服务，包含以下特性：  
> - 集群中心化配置  
> - 自动负载均衡和fail-over特性  
> - 集成zookeeper实现集群协调和配置  

> solr cloud灵活的分布式引擎和搜索，不需要master节点来分配子节点、分片、备份等，而是通过zookeeper来实现服务发现、配置。  

首先需要确保机器已经安装jdk1.7以上版本，然后安装solr和zookeeper服务，具体的安装配置步骤在对应的官网文档是写的很清楚，这里就不再赘述了。

<!--more-->

# solr常用操作

## 停止solr服务

1. 按指定port停止
```bash
cd /data/xxx/solr/solr/bin
./solr stop -p <port>
```
2. 全部停止
```bash
cd /data/xxx/solr/solr/bin
./solr stop -all
```

## 启动solr服务

```bash
cd /data/xxx/solr/solr/bin
./solr start -cloud -m 1g -z <zookeeper list> -s <solr server dir> -h <solr host> -p <solr port>
./solr start -cloud -m 1g -z 10.25.66.82:2181,10.25.66.82:2182,10.25.65.79:2181,10.25.65.78:2181 -s /data/xxx/solr/solr_data/node1 -h 10.25.66.82 -p 8983
./solr start -cloud -m 1g -z 10.25.66.82:2181,10.25.66.82:2182,10.25.65.79:2181,10.25.65.78:2181 -s /data/xxx/solr/solr_data/node2 -h 10.25.66.82 -p 8984
./solr start -cloud -m 1g -z 10.25.66.82:2181,10.25.66.82:2182,10.25.65.79:2181,10.25.65.78:2181 -s /data/xxx/solr/solr_data/node1 -h 10.25.65.79 -p 8983
./solr start -cloud -m 1g -z 10.25.66.82:2181,10.25.66.82:2182,10.25.65.79:2181,10.25.65.78:2181 -s /data/xxx/solr/solr_data/node2 -h 10.25.65.78 -p 8983
```

## 新建core/collection

```bash
cd /data/xxx/solr/solr/bin
./solr create -c <name> -d <config dir> -n <config name> -shards <shards num> -replicationFactor <replica per shard>
./solr create -c mcservers -d /data/xxx/solr/solr_data/configset/mcservers -n mcservers -shards 1 -replicationFactor 2
./solr create -c account -d /data/xxx/solr/solr_data/configset/account -n account -shards 2 -replicationFactor 2
#Create a core or collection depending on whether Solr is running in standalone (core) or SolrCloud mode (collection)
```

> 这里的shards参数依据你的数据量来定，数据量不大的话不建议创建太多分片（太多反而影响query效率），replicationFactor一般2到3个备份即可。

## 删除core/collection

```bash
cd /data/xxx/solr/solr/bin
./solr delete -c <name> -deleteConfig <true/false>
#删除指定collection,deleteConfig默认true,从zookeeper中删除collection对应的配置信息
```

## 检查本机solr服务状态

```bash
cd /data/xxx/solr/solr/bin
./solr status
```

## solr健康检查

```bash
cd /data/xxx/solr/solr/bin
./solr healthcheck -z 10.25.65.79:2182 -c mcservers
#任意指定一个zookeeper实例
```

## 上传collection配置文件到zookeeper

```bash
cd /data/xxx/solr/solr/bin
./solr zk -upconfig -n <config name> -d <configset dir> -z <zookeeperHost>
```

## 更新配置

```bash
cd /data/xxx/solr/solr/server/scripts/cloud-scripts
./zkcli.sh -z 10.25.66.82:2181 -cmd upconfig -confname mcservers -confdir /data/xxx/solr/solr_data/configset/mcservers/conf
./zkcli.sh -z 10.25.66.82:2181 -cmd upconfig -confname account -confdir /data/xxx/solr/solr_data/configset/account/conf
#清除配置
./zkcli.sh -z 10.25.65.78:2181 -cmd clear /configs
http://xxx.yyy.com/solr/mcservers/dataimport?command=reload-config
http://xxx.yyy.com/solr/mcservers/dataimport?command=full-import&clean=true&commit=true
```

## 从zookeeper下载collection的配置文件

```bash
cd /data/xxx/solr/solr/bin
./solr zk -downconfig -n <config name> -d <dest dir> -z <zk host>
```

## 使用中文分词库ik-analyzer

```bash
wget https://github.com/EugenePig/ik-analyzer-solr5/archive/master.zip
mvn clean install
#ik-analyzer自带了stopword.dic,ext.dic会影响分词效果，可以清空这两个文件然后编译
#编译慢的话切换下mvn repos http://maven.oschina.net/help.html
```

1. 将ik-analyzer-solr5-5.x.jar放到/data/xxx/solr/solr/dist/目录中
2. 加入到solr classpath, configset/mcservers/conf/solrconfig.xml中添加<lib dir="${solr.install.dir:../../../..}/dist/" regex="ik-analyzer-solr5-5.x.jar" />

## 关于field定义

1. indexed 表示需不需要建立索引，以便之后对这个field进行检索
2. stored 表示要不要随索引一起存储这个field本身的值，以便查询时直接获取该值（一般值较小的话可以设置stored为true，太大的话会导致索引过大）
3. omitNorms 字段的长度不影响得分和在索引时不做boost时，设置它为true。一般文本字段设置为false。
4. termVectors 如果字段被用来做more like this 和highlight的特性时应设置为true。

## solr时区

solr默认UTC时区，可以在/data/xxx/solr/solr/bin/solr.in.sh中设置SOLR_TIMEZONE="GMT+8:00"

## zookeeper操作

1. 启动
```bash
./zkServer.sh start ../conf/zoo1.cfg
```
2. 停止
```bash
./zkServer.sh stop ../conf/zoo1.cfg
```

# 关键配置

**data-config.xml**

```xml
<dataConfig>
  <dataSource type="JdbcDataSource"
              driver="com.mysql.jdbc.Driver"
              url="jdbc:mysql://url:port/dbname"
              user="user"
              password="password"
              batchSize="-1"
              readOnly="true"/>
  <document name="docname">
    <entity name="docname"
            pk="id"
		        query="select id,name from tb where xxx>0"
            deltaImportQuery="select id,name from tb where id='${dataimporter.delta.id}'"
            deltaQuery="select id from tb where status>0 and updateAt >= '${dataimporter.last_index_time}'">
		<field column="id" name="id" />
		<field column="name" name="name" />
    </entity>
  </document>
</dataConfig>
```

其中`query`用于full-import,`deltaImportQuery`和`deltaQuery`用于delta-import。  
dateimport组件会在每次import之后更新zookeeper节点中的`dateimport.properties`文件，`${dataimporter.last_index_time}`就是从这里取值的。

**managed-schema**

```xml
<schema name="docname" version="1.6">
  <!--add by wangpeng begin搜索字段-->
	<!-- 支持类似mysql的like 查询,比如:输入"Exp"可以搜到:"似懂非懂分ExplodingTNT345惊愕斯蒂芬森的"
    <fieldType name="text_general_edge_ngram" class="solr.TextField" positionIncrementGap="100">
       <analyzer type="index">
          <tokenizer class="solr.StandardTokenizerFactory"/>
          <filter class="solr.StopFilterFactory" ignoreCase="true" words="stopwords.txt" enablePositionIncrements="true" />
          <filter class="solr.WordDelimiterFilterFactory"
                generateWordParts="1"
                generateNumberParts="1"
                catenateWords="0"
                catenateNumbers="0"
                catenateAll="0"
                preserveOriginal="1"
                />
          <filter class="solr.EdgeNGramFilterFactory" minGramSize="1" maxGramSize="50" side="front"/>
          <filter class="solr.LowerCaseFilterFactory"/>
       </analyzer>
       <analyzer type="query">
          <tokenizer class="solr.StandardTokenizerFactory"/>
          <filter class="solr.SynonymFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true"/>
          <filter class="solr.LowerCaseFilterFactory"/>
       </analyzer>
    </fieldType>-->
    <field name="id" type="long" indexed="true" stored="true" required="true" multiValued="false" />
    <field name="_version_" type="long" indexed="true" stored="true"/><!--solr 默认的-->
    <field name="_root_" type="string" indexed="true" stored="false"/><!--solr 默认的-->
    <field name="name" type="text_chinese" indexed="true" stored="true" omitNorms="false" termVectors="true" />
    <!--<field name="name_like" type="text_general_edge_ngram" indexed="true" stored="false" omitNorms="false" termVectors="true" />-->

    <field name="_text_" type="text_chinese" indexed="true" stored="false" multiValued="true" omitNorms="false" termVectors="true" />
    <copyField source="*" dest="_text_"/>
    <defaultSearchField>_text_</defaultSearchField>
    <solrQueryParser defaultOperator="AND"/>
  <!--ik中文分词-->
  <fieldType name="text_chinese" class="solr.TextField" >
	  <analyzer type="index" useSmart="false" class="org.wltea.analyzer.lucene.IKAnalyzer"/>
      <analyzer type="query" useSmart="true" class="org.wltea.analyzer.lucene.IKAnalyzer"/>
  </fieldType>
  <!--add by wangpeng end-->
  <uniqueKey>id</uniqueKey>
```

# solr安全性

solr cloud构建之后默认会有一个web站点用于查看solr集群信息和管理，但是目前并没有身份验证的机制，所以是完全开放的，任何人都能访问到你的solr集群站点，这样存在安全性问题。我们的解决方案是限制服务器内网可访问，如果工作环境需要访问的话，单独设置工作环境ip的访问权限。还有，solr也不应该直接对外提供服务，而是通过构建自己的搜索api来对外提供服务（搜索api跟solr交互，相当于搜索api做了一层代理），因为直接对外服务同样存在安全问题，比如随意删除你的collection等。
