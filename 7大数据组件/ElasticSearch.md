### ElasticSearch

<https://www.bilibili.com/video/av71202391>

#### 1.简介

**Elasticsearch是一个开源的分布式、RESTful 风格的搜索和数据分析引擎**，它的底层是开源库Apache Lucene。
   Lucene 可以说是当下最先进、高性能、全功能的搜索引擎库——无论是开源还是私有，但它也仅仅只是一个库。为了充分发挥其功能，你需要使用 Java 并将 Lucene 直接集成到应用程序中。 更糟糕的是，您可能需要获得信息检索学位才能了解其工作原理，因为Lucene 非常复杂。
   为了解决Lucene使用时的繁复性，于是Elasticsearch便应运而生。它使用 Java 编写，内部采用 Lucene 做索引与搜索，但是它的目标是使全文检索变得更简单，简单来说，就是对Lucene 做了一层封装，它提供了一套简单一致的 RESTful API 来帮助我们实现存储和检索。
   当然，Elasticsearch 不仅仅是 Lucene，并且也不仅仅只是一个全文搜索引擎。 它可以被下面这样准确地形容：

- 一个分布式的实时文档存储，每个字段可以被索引与搜索；
- 一个分布式实时分析搜索引擎；
- 能胜任上百个服务节点的扩展，并支持 PB 级别的结构化或者非结构化数据。

由于Elasticsearch的功能强大和使用简单，维基百科、卫报、Stack Overflow、GitHub等都纷纷采用它来做搜索。现在，Elasticsearch已成为全文搜索领域的主流软件之一。

**ES和Solr的区别：**

- 配置文件方面：ES配置和使用简单
- 分布式协调方面：ES自带，Solr用的是zk
- 文件格式方面：ES只支持json，Solr支持的格式多
- 数据可视化方面：ES使用Kibana，Solr自带
- 索引更新方面：ES更新索引快，Solr查询快，但是索引更新慢
- 成熟度与社区量：ES版本更新快，Solr比较成熟，社区量大

##### 1.1 搜索原理

将每个document进行分词，分词后根据B+树建立倒排索引，每个词都有一个posting，其中记录着id，词频（用来对document进行打分），位置和偏移量（用来快速找到在document找那个的位置）

- B+树演示

  <https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html>

##### 1.2 分词过程

1. Character Filter

   过滤掉一些特殊字符

2. Tokenizer

   分词

3. Token Filters

   过滤没有用的词

例如：content="有笔记吗up主,有笔记嘛？,有课件吗"

经过Character Filter：有笔记吗up主有笔记嘛有课件吗

经过Tokenizer：有 笔记 吗 up主 有 笔记 嘛 有 课件 吗

经过Token Filters：笔记（词频、位置、偏移量） up主（词频、位置、偏移量） 课件（词频、位置、偏移量）

##### 1.3 常见分词器

- 中文分词器

  | 名称   | 介绍                                   | 特点                                 | 地址                                                    |
  | ------ | -------------------------------------- | ------------------------------------ | ------------------------------------------------------- |
  | IK     | 实现中英文单词切分                     | 自定义词库                           | https://github.com/medcl/elasticsesarch-analysis-ik     |
  | jieba  | python流行分词系统，支持分词和词性标注 | 支持繁体、自定义、并行分词、词性标注 | https://github.com/singlee/elasticsearch-jieba-plugin   |
  | Hanlp  | java工具包                             | 普及NLP在生产环境中的应用            | https://github.com/hankcs/HanLP                         |
  | THULAC | 清华大学中文词法分析工具包             | 可词性标注                           | https://github.com/microbun/elasticsearch-thulac-plugin |

##### 1.4 IK分词器

分为ik_smart和ik_max_word两种，ik_smart是最少非常，ik_max_word是最细粒度分词。

##### 1.5 自定义分词器

<https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-overview.html>

##### 1.6 Elastic Stack

<https://baijiahao.baidu.com/s?id=1644219403279330326&wfr=spider&for=pc>

#### 2.安装

os: linux6.x

##### 2.1 环境准备

- 增大系统上部署软件的内存和硬盘

  ```shell
  vim /etc/security/limits.conf
  * soft nproc 655350
  * soft nofile 655350
  * hard nproc 655350
  * hard nofile 655350
  ```

- 配置系统最大线程数

  ```shell
  vim /etc/sysctl.conf
  vm.max_map_count=262144
  ```

- 配置用户最大线程数

  ```shell
  vim /etc/security/limits.d/90-nproc.conf
  *	soft	nproc	4096
  root	soft	nproc	unlimited
  ```

- 生效

  ```shell
  sysctl -p
  ```

- ES不能使用root用户启动

  ```shell
  user esuser
  password esuser
  chown -R esuser /es安装目录
  
  ```

##### 2.2 修改ES配置文件

- elasticsearch.yml

  ```yaml
  cluster.name: my-cluster
  node.name: node-1
  path.data: /ES节点信息数据存放路径
  path.logs: /ES日志存放路径
  bootstrap.memory_lock: false
  # 如果操作系统是centos6需要配置
  bootstrap.system_call_filter: false
  network.host: 192.168.0.1
  discovery.zenping.unicast.hosts: ["127.0.0.1"]
  ```

##### 2.3 启动ES

```shell
bin/elasticsearch
```

验证是否启动成功：

在浏览器中地址栏中输入	ip:9200

#### 3.配置IK分词器

- ik分词器版本应该和es版本一致

- 将压缩包放在plugins目录下，并解压  unzip ...

#### 4.安装kibana

- 修改配置文件src/config/kibana.yml

  ```yaml
  server.prot: 5601
  server.host: "192.168.0.1"
  elasticsearch.url: "http://192.168.0.2:9200"
  ```

- 启动kibana   bin/kibana

- 验证是否启动成功：在浏览器中地址栏中输入	ip:5601

#### 5.使用kibana操作ES

```json
# 查询ES集群详情
GET _cat/
# 查询节点健康状态
GET _cat/health
# 查询ES集群节点详情
GET _cat/nodes?v

# 查询index详情
GET _cat/indices?v
# 创建index
PUT test_index1
# 创建type（PUT：自定义type；POST：按照指定格式创建type）
PUT /test_index2
{
	"mappings": {
        "test_type1": {
            "properties": {
                "username": {"type": "text"},
                "password": {"type": "long"},
                "age": {"type": "integer"}
            }
        }
    }
}
POST /test_index3/test_type1
{
    "properties": {
        "username": {"type": "text"},
        "password": {"type": "long"},
        "age": {"type": "integer"}
    }
}
# 查询所有
GET /index/type/1?source=name,age
# 查询某个字段
GET /index/type_search?
# 查询type
GET /test_index3/_mapping/test_type1
# 向type中添加document（PUT必须指定id；POST不用指定id，会随机生成UUID）
PUT /test_index3/test_type1/20
{
    "username": "zhangsan",
    "password": 132465,
    "age": 18
}
POST /test_index3/test_type1
{
    "username": "lisi",
    "password": 132465,
    "age": 18
}
# 查询指定的document
GET /test_index3/test_type1/20
# 修改指定的document
POST /test_index3/test_type1/20/_update
{
    "doc": {
        "password": 654321
    }
}
# 删除指定的document
DELETE /test_index3/test_type1/20
```

#### 6.ES mapping

定义数据库中表的结构，通过mapping来控制索引存储数据的设置

- 定义Index下的Field Name
- 定义字段的类型
- 定义倒排索引的相关配置，如id，词频，position，offset

1. **mapping中的字段类型一旦设定后禁止修改**

2. **获取索引mapping：**

   GET /index/_mapping

3. **Dynamic Mapping :**

    - true 允许新增字段
    - false 不允许新增字段，可以插入字段但查询不出来
    - strict 不允许新增字段，插入字段会报错

#### 7.ES search

- 多个查询

  GET /index_1,index_2/_search

- 匹配查询

  GET /index\_*/\_search

- 参数查询

  GET /index/_search?q=keyword&df=user&sort=age:asc&from=4&size=10&timeout=10s

- 泛查询 alfred

  GET /index/type/_search?q=aa

- 指定字段分词查询 phrase

  GET /index/type/_search?q=username:aa bb

- 指定字段查询 term

  GET /index/type/_search?q=username:"aa bb"

- 组查询

  GET /index/type/_search?q=username:(aa or bb)

- 展示查询详情

  GET /index/type/_search?q=username:(aa or bb) and cc

  {"profile":true}

- 支持 逻辑与或非，正则，>=，模糊（a~2：匹配包含a的三个字母）

```reStructuredText
GET index/_search
{
	"query":{
		"match":{
			"query":"aa bb",
			"operator":"and"
		}
	}
}
# 在多字段中查询
GET index/_search
{
	"query":{
		"query_string":{
			"fields":["username","job"],
			"query":"aa OR (bb AND cc)"
		}
	}
}
# 可空缺多少个词（slop）
GET index/_search
{
	"query":{
		"match_phrase":{
			"job":{
				"query":"aa cc",
				"slop":1
			}
		}
	}
}
# 范围查询
GET index/_search
{
	"query":{
		"range":{
			"age":{
				"gte":10,
				"lte":20
			}
		}
	}
}
# bool查询(rank)
GET index/_search
{
	"query":{
		"bool":{
			# 文档中必须符合must中所有条件，会影响得分
			"must":{
				...
			},
			# 文档中必须不包含must_not中任意条件，会影响得分
			"must_not":{
				...
			},
			# 文档中可以符合should中的条件，会影响得分
			"should":{
				...
				"minmum_should_match":1
			},
			# 只过滤符合条件的文档，不影响得分
			"filter":{
				...
			}
		}
	}
}
```

#### 8.相关性算分

- 影响得分的因素

  1. Term Frequency（TF）：词频，即单词在文档中出现的次数
  2. Document Frequency（DF）：文档词频，即单词出现的文档数
  3. Inverse Document Frequency（IDF）：逆向文档词频，即IDF=1/DF，如果单词在所有文档中出现的越少，证明单词越重要
  4. Field-length Norm：文档越短，相关度越高

- 相关性得分算法

  - IT/IDF模型

  - BM25模型（5.X之后）

#### 9.集群

- 集群搭建

- 集群监控

  cerebro

- 集群特点

  master管理其他节点信息和index信息

- 集群状态

  | 状态   | 解释                             |
  | ------ | -------------------------------- |
  | green  | 所有主分片和副本分片都可用       |
  | yellow | 所有主分片可用，副本分片部分可用 |
  | red    | 部分主分片可用                   |

- 路由（事务，ack机制）

#### 10.Logstash

##### 10.1 input

- **stdin** 可以从管道输入，也可以从终端输入

  通用配置：

  1. codec：类型为codec
  2. type：类型为string，自定义事件类型，用于filter判断
  3. tags：类型为array，自定义事件标签，用于filter判断
  4. add_field：类型为hash，为事件添加字段

  启动：

  ```shell
  echo helloword | ./bin logstash -f ./myconf/test.conf
  ```

  ````reStructuredText
  input{
  	stdin{
  		codec=>"plain"
  		tags=>["tag1","tag2"]
  		type=>"type1"
  		add_field =>{"key"=>"value"}
  	}
  }
  output{
  	stdout{
  		codec=>"rubydebug"
  	}
  }
  ````

- http

  ```reStructuredText
  input{
  	http{port => 7474}
  }
  ```

  

- file** 读取文件

  通用配置：

  1. path=>["/var/log/aa/aa.log","/var/log/bb/bb*.log"]  文件位置
  2. exclue=>"*.gz" 不读取哪些文件
  3. sincedb_path=>"/var/log/message"  记录sincedb文件路径
  4. start_postion=>"begining" 从文件开始读
  5. stat_interval=>1000 单位秒，定时检查文件是否有更新，默认1s

  ```text
  input{
  	file{
  		path=>["/var/log/aa/aa.log","/var/log/bb/bb*.log"]
  		start_postion=>"begining"
  		type=>"type2"
  	}
  }
  output{
  	stdout{
  		codec=>"rubydebug"
  	}
  }
  ```

- **elasticsearch**

  ```reStructuredText
  input{
  	elasticsearch{
  		hosts => "192.168.0.1"
  		index => "index1"
  		query => '{"query":{match_all:{}}}'
  	}
  }
  output{
  	stdout{
  		codec=>"rubydebug"
  	}
  }
  ```

##### 10.2 filter

data：日期解析

grok：正则匹配解析

dissect：分割符解析

mutate：对字段做处理，如重命名、删除、替换

json：按照json解析字段

geoip：增加地理位置数据

ruby：利用ruby代码来动态修改event

#### 11.使用Java API操作ES

- pom.xml

  ```xml
  <dependency>
      <groupId>org.elasticsearch.client</groupId>
      <artifactId>transport</artifactId>
      <version>6.4.0</version>
      <exclusions>
          <exclusion>
              <groupId>org.elasticsearch</groupId>
              <artifactId>elasticsearch</artifactId>
          </exclusion>
      </exclusions>
  </dependency>
  ```

- application.properties

  ```properties
  spring.elasticsearch.ip=192.168.0.1
  # ES的java api端口号
  spring.elasticsearch.prot=9300
  spring.elasticsearch.clusterName=my-cluster
  spring.elasticsearch.nodeName=node1
  spring.elasticsearch.pool=5
  ```

- 创建配置类

  ESProperties.java

  ```java
  @ConfigurationProperties(perfix="spring.elasticsearch")
  @Component
  @Data
  class ESProperties{
      private String ip;
      private Integer prot;
      private String clusterName;
      private String nodeName;
      private Integer pool;
  }
  ```

  ESConfig.java

  ```java
  @Configuration
  class ESConfig{
      @Autowired
      private ESproperties esProperties;
      
      @Bean("transportClient")
      public TransportClient getTransportClient(){
          TransportClient transportClient = null;
          try{
              Settings settings = Settings.bulider()
                  .put("cluster.name", esProperties.getClusterName())
                  .put("node.name", esProperties.getNodeName())
                  .put("client.transport.sniff", true) //如果有新的节点进入集群中，会自动识别
                  .put("thread_pool.search.size", esProperties.getPool())
                  .build()
              transportClient = new PreBuiltTransportClient(settings);
              TransportClient address = new TransportAddress(
                  esProperties.getIp(),esProperties.getPort())
              transportClient.addTransportAddress(address)   
          }catch(Exception e){
              e.printStackTrace();
          }
          return transportClient;
      }
  }
  ```

- 创建ES工具类

  ESUtils.java

  ```java
  @Component
  class ESUtils{
      private static TransportClient client;
      @Autowired
      private TransportClient transportClient;
      
      @PostConstruct
      public void init(){
          client = this.transportClient
      }
      
      public static boolean isIndexExist(String index){
          IndicesExistsResponse resp = client.admin().indices().exists(new IndicesExistsRequest(index).actionGet());
          return resp.isExists();
      }
      
      public static boolean createIndex(String index){
          if (isIndexExist(index)){
              return false;
          }
          CreateIndexPresponse resp = client.admin().indeices().prepareCreate(index).execute().actionGet();
          return createIndexResponse.isAcknowledged()
      }
  }
  ```

  
