---
layout:     post
title:      "ElasticSearch学习记录"
subtitle:   "ElasticSearch"
date:       2017-09-26 00:30:00
author:     "Sean"
header-img: "img/post-bg-android.jpg"
catalog: true
tags:
    - ElasticSearch
---

> ES在生产环境中到底分配多少内存，JVM Heap到底给分配多大?

---
> ### 如何分配Heap？
> * heap 不能超过32G （参数：ES_HEAP_SIZE）
为什么不能超过32G？因为目前大部分程序都是运行在64位的机器上，而且HotSpot vm内部Java对象表示的长度从32位变成64位，即普通对象指针(Ordinary Object Points or oops).这也就意味着，在64位的机器上比在32位的机器上运行的程序所占用的内存要大于1.5倍左右。所以必须通过指针压缩的技术把地址长度变小，从而提高了运行程序在64位的机器上的性能。

零基压缩技术会根据堆内存的大小以及平台特性来选择不同的策略：
堆内存小于4Gb，直接使用压缩对象指针进行寻址，无需压缩和解压;
堆内存大于4Gb，则尝试分配小于32Gb的堆内存，并使用零基压缩技术；
如果仍然失败，则使用普通的对象指针压缩技术，即narrow-oop-base。

> * 剩余的mem留给lucene
> * 确保启动log里面包含“heap size [27gb], compressed ordinary object pointers [true]”，确保已经使用了对象指针压缩技术。
> *  关闭swap (建议这样设置 vm.swappiness = 1)
> * 开始mlockall（bootstrap.mlockall: true）

---
## Indexing Performance Tips
> 索引性能可以根据具体的应用场景而定，如果是ELK场景，一般需要indexing 性能非常好，搜索几秒钟是可接受的，如果是面对用户的搜索，必须毫秒级别返回。

## Test Performance Sicentifically
> 准确的性能测试是比较困难的，如何科学有效的测试以及优化是很重要的。不能随便的调整参数，需要有比较，调整参数为最佳值。一般一个合理的步骤如下：
>
> * 在单个节点、用一个shard，没有复本。
> * 保证100%默认配置，测试并记录结果
> * 性能压测一般至少30分钟，你需要评估长时间的性能，而不是短暂的spike or latencies. 哪些可能是GC或者segments merges.
> * 记录修改过的单个配置，严格的测试，如果修改的某个参数对性能有改善，就保持下来，继续测试下一个！

--
## Bluk 大小设置
* 建议每次bluk：5-15MB
* 如果中如果出现*EsRejectedExecutionException*，代表server建立索引已经达到瓶颈了。
* 从硬件层优化，使用ssd或者PCI-E

---
## Segments and Merfeging

###关闭merge throttling

>PUT /_cluster/settings
{
    "transient" : {
        "indices.store.throttle.type" : "none"
    }
>}

### 开启merge throttling

>PUT /_cluster/settings
{
    "persistent" : {
        "indices.store.throttle.max_bytes_per_sec" : "100mb"
    }
>}

---

## 搜索
### 全文检索
所有的查询都要计算相关度，不是所有的查询都有分析阶段。ES中查询分为两大类：

* 基于短语(Term-based)查询：
这类查询主要用于精确匹配，他是一个底层的查询操作，没有分析阶段，这些查询单一在短语上执行，查询倒排索引。例如查询单词"elasticsearch"，如果使用term 去查询，会到倒排索引中搜索精确的"elasticsearch"而不是"Elasticsearch"等其他的形式。如果想把ES当成数据库使用，就必须使用term去搜索。

* 全文(Full-text)检索：
这是一个高级的查询操作，例如"query" or "query_string",会对字段进行分析。

1. 如果查询的是"date"或"integer"字段，他们会识别查询是日期或者整数格式
2. 如果检索到一个字段是准确值(not_analyzed),他们会把整个查询做为一个短语
3. 如果查询的是一个全文字段，查询会先用到解析器解析语句，生产需要的查询短语列表，然后对列表中每个查询执行低级短语查询，合并查询结果，得到最终的结果。

> 提示：
> 精确的查询，还是需要使用过滤器，单一的查询相当于是/否的问题，过滤器可以更好的描述这一类问题，并且过滤器有cache可以提升性能。

```
GET /_search
{
    "query": {
        "filtered": {
            "filter": {
                "term": { "gender": "female" }
                }
        }
}
```



## 索引管理
### 创建索引
更新索引mapping

```
 PUT /my_index
{
    "settings": { ... any settings ... },
    "mappings": {
        "type_one": { ... any mappings ... },
        "type_two": { ... any mappings ... },
        ...
}
```

防止自动创建索引：

> ￼action.auto_create_index: false

### 删除索引
例如：

> DELETE /target_index
> DELETE /target_index1,target_index2
> DELETE /_all


### 索引设置
设置索引的分片数量
```
PUT /my_temp_index
{
    "settings": {
        "number_of_shards" :   1,
        "number_of_replicas" : 0
}
}
```
默认分片：5 primary, 1 slave;可以动态调整slave分配数量

```
PUT /my_index/_settings
{
  "number_of_replicas": 1
}
```

### 分析器

分析器(analyzer) 个人理解应该是在建立索引的时候，对文本的处理。
分析器会按照如下三个组件的顺序完成分析

*字符过滤器 (Character filters)*
> 字符过滤器是让字符串在被分词前变得更加“整洁”。例如,如果我们的文本是 HTML 格 式,它可能会包含一些我们不想被索引的 HTML 标签,诸如 <p> 或 <div> 。
我们可以使用 html_strip 字符过滤器 来删除所有的 HTML 标签,并且将 HTML 实体转 换成对应的 Unicode 字符,比如将 &Aacute; 转成 Á 。
> 一个分析器可能包含零到多个字符过滤器。

*分词器 (Tokenizers)*
> 一个分析器 必须 包含一个分词器。分词器将字符串分割成单独的词(terms)或标记 (tokens)。 standard 分析器使用 standard 分词器将字符串分割成单独的字词,删除 大部分标点符号,但是现存的其他分词器会有不同的行为特征。
例如,keyword 分词器输出和它接收到的相同的字符串,不做任何分词处理。 [whitespace 分词器]只通过空格来分割文本。[pattern >分词器]可以通过正则表达式来 分割文本。

*标记过滤器 (Token filters)*
>分词结果的 标记流 会根据各自的情况,传递给特定的标记过滤器。
标记过滤器可能修改,添加或删除标记。我们已经提过 lowercase 和 stop 标记过滤 器,但是 Elasticsearch 中有更多的选择。 stemmer 标记过滤器将单词转化为他们的根 形态(root form)。 ascii_folding 标记过滤器会删除变音符号,比如从 très 转为
>tres 。 ngram 和 edge_ngram 可以让标记更适合特殊匹配情况或自动完成。

---

## 集群分片分配

> ES集群中节点的分片主要是通过一个被称为*Shard allocation*的东西去分配，它做了如下工作：

> * 初始化分片
> * 副本分配
> * 数据平衡
> * 节点加入或者移除

#####  分配参数详细
这些参数可用于控制分片的分配和恢复

* *cluster.routing.allocation.enable*
> 开启或者关闭特定种类类型
>   all (default) 默认允许分配所有种类的分片
>   primaries 仅允许为主分片分配分片
>   new_primaries 仅允许为新索引的主分片分配分片
>   none 不为任何类型的分片分配分片
> 当一个节点重启的时候，这些配置不会影响节点从本地恢复主分片。一个节点重启，这个节点上主分片对应的副本将立马变成主分片，但是前提是要满足 [index.recovery.initial_shards][link_id1]设置的值。

[link_id1]: https://www.elastic.co/guide/en/elasticsearch/reference/2.3//index-modules.html#index.recovery.initial_shards "index.recovery.initial_shards"
* *cluster.routing.allocation.cluster_concurrent_rebalance
> 这个参数控制集群数据平衡分片的[并发数][link_id2]

[link_id2]: https://www.elastic.co/guide/en/elasticsearch/reference/2.3/shards-allocation.html "cluster.routing.allocation.cluster_concurrent_rebalance"

* *cluster.routing.allocation.node_concurrent_recoveries*
> 除了主分片恢复任务以外， 在一个节点上同时允许多少个副分片去恢复，1.6开始冷数据可用从本地恢复 (默认值：2)

* *cluster.routing.allocation.node_initial_primaries_recoveries*
> 当一个节点重启时，允许同时恢复几个主分片 (默认值: 4)

* *cluster.routing.allocation.same_shard.host*
> 基于主机名和地址，防止在单个物理机上多实例中分配同一个分片，默认是 false,这个参数主要在多个实例部署在一个节点上的情况有效

* *indices.recovery.concurrent_streams*
> 这个参数主要用于控制节点从网络恢复副本分片到本地时的数据流个数，默认是3个。

* *indices.recovery.concurrent_small_file_streams*
> 这个参数用于控制从网络中恢复分片的个数，主要针对小于5M的小文件。

##### 索引策略
下面的这些策略可以用于去管理分片恢复

* *indices.recovery.max_bytes_per_sec*
> 用来控制节点恢复是的速率
>  Default：40mb

* *indices.recovery.concurrent_streams*
> Defaults to 3.

* *indices.recovery.concurrent_small_file_streams*
> Defaults to 2.

* *indices.recovery.file_chunk_size*
> Defaults to 512kb.

* *indices.recovery.translog_ops*
> Defaults to 1000.

* *indices.recovery.translog_size*
> Defaults to 512kb.

* *indices.recovery.compress*
> Defaults to true.


##### 快速完成副本复制
```
PUT /_cluster/settings
{
  "transient": {
  "indices.recovery.max_bytes_per_sec": "500mb",
  "indices.recovery.concurrent_streams": "20",
  "cluster.routing.allocation.node_concurrent_recoveries": "20",
  "cluster.routing.allocation.cluster_concurrent_rebalance": "20"
  }
}

```
> 提示：以上操作尽量谨慎，不要影响到线上运行的业务，副本恢复完成以后，记得要修改成默认值。
