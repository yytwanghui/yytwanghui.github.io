---
title: 08_SpringBoot与检索
date: 2020-11-17 18:21:51
categories:
- SpringBoot
tags:
- SpringBoot
- 检索
---

<center><font size=4 color="red">08_SpringBoot与检索</font></center>

<!--more-->

# SpringBoot与检索

## ElasticSearch

ElasticSearch是基于Lucene做了一些封装和增强

**Lucene**：就是一个封装好的jar包，基于java写的，是一个信息检索工具，不包含搜索引擎。包含：索引结构；读写索引的工具；排序，搜索规则等

ElasticSearch的目的就是为了快速索引

ElasticSearch的结构为，以和数据库为例：

| 名称 | 相当于数据库的 |
| ---- | -------------- |
| 索引 | 库             |
| 类型 | 表             |
| 文档 | 每一条的数据   |
| 属性 | 属性           |



## 安装

使用docker-compose同时安装ElasticSearch，es-head，kibana，并在ElasticSearch中添加ik分词器

#### 安装前的准备

**elasticsearch.yml**

目录位置在./elasticsearch/config/elasticsearch.yml，这里的根目录是docker-compose.yml所在的目录

也即使：/home/yytwanghui/docker/es-kibana

![](根目录.jpg)

```yaml
discovery.type: "single-node"
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
transport.tcp.port: 9300
path.logs: /usr/share/elasticsearch/logs
#http.cors配置跨域
http.cors.enabled: true
http.cors.allow-origin: "*"
xpack.security.audit.enabled: true
```

**kibana.yml**

目录位置在./kibana/config/kibana.yml

```yaml
# 服务端口
server.port: 5601
# 服务IP
server.host: "0.0.0.0"
# ES
elasticsearch.hosts: ["http://192.168.31.181:9200"]
# 汉化
i18n.locale: "zh-CN"
```

**docker-compose.yml**

```yaml
version: "3"
services:
  # 配置elasticsearch
  elasticsearch:
    image: elasticsearch:7.6.1
    container_name: elasticsearch
    restart: always
    environment:
      - "cluster.name=elasticsearch" #设置集群名称为elasticsearch
      - "discovery.type=single-node" #以单一节点模式启动
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m" #设置使用jvm内存大小,如果内存不大，这里必须设置，不设置默认2G
    volumes:
      - "./elasticsearch/data:/usr/share/elasticsearch/data:rw"
      - "./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
      - "./elasticsearch/logs:/user/share/elasticsearch/logs:rw"
      - "./elasticsearch/plugins/ik:/usr/share/elasticsearch/plugins/ik"
    ports:
      - 9200:9200
      - 9300:9300
 
  # 配置es-head
  es-head:
    container_name: es-head
    image: mobz/elasticsearch-head:5
    restart: always
    ports:
      - 9100:9100
    depends_on:
      - elasticsearch
 
  # 配置kibana
  kibana:
    container_name: kibana
    hostname: kibana
    image: kibana:7.6.1
    restart: always
    ports:
      - 5601:5601
    volumes:
      - "./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml"
    environment:
      - elasticsearch.hosts=http://192.168.31.181:9200
    depends_on:
      - elasticsearch
```

**拉去镜像，启动容器**

```powershell
docker-compose build
docekr-compose up -d
```

**查看启动情况**

```yaml
docker ps   # 查看启动的容器，发现ES并不是up状态
docker logs es容器id  # 查看日志
```

发现报错：

```powershell
java.nio.file.AccessDeniedException: /usr/share/elasticsearch/data/nodes
```

解决方法：

```powershell
chmod 777 /home/yytwanghui/docker/es-kibana/elasticsearch/data
```

#### 注意事项

* 需要安装nodejs

  ```powershell
  sudo apt update
  sudo apt install nodejs
  sudo apt install npm
  ```

* 如果没有配置ik分词器，就不要在数据卷里指定ik的插件，否则找不到ik的配置文件会报错

* 如果报错：

  ```powershell
  java.nio.file.AccessDeniedException: /usr/share/elasticsearch/data/nodes
  ```

  就给需要挂载的data增加权限

  ```powershell
  chmod 777 /home/yytwanghui/docker/es-kibana/elasticsearch/data
  ```

* 启动时报错：

  ```powershell
  Unknown properties in plugin descriptor: [jvm, site, isolated]"
  ```

  这是因为ik分词器版本太低，换成7.6.1版本就OK了
  
  其实只要保证ES、kibana、es-head版本完全一样即可
  
* es-head里无法查询到具体的文档信息

  ![](es-head-error.jpg)

  解决办法：

  ```powershell
  # 进入容器
  docker exec -it es-head容器id /bin/bash
  # 在容器内更新apt-get
  apt-get update
  # 在容器内安装vim
  apt-get install vim
  # 修改vendor.js文件
  vim _site/vendor.js
  ```

  修改内容如下：

  ```js
  第6886行 : contentType: "application/x-www-form-urlencoded
  改为 contentType: “application/json;charset=UTF-8”
  第7573行: var inspectData = s.contentType === “application/x-www-form-urlencoded” &&
  改为 var inspectData = s.contentType === “application/json;charset=UTF-8” &&
  ```

  然后重启容器

  ```powershell
  docker-compose restart
  ```

  

#### 访问

ES的访问：http://192.168.31.181:9200/

![](es.jpg)

es-head的访问：http://192.168.31.181:9100/

这里只所以可以跨域是因为**elasticsearch.yml**配置了

```yaml
http.cors.enabled: true
http.cors.allow-origin: "*"
```

![](es-head.jpg)

kibana的访问：http://192.168.31.181:5601/

这里只所以可以显示中文，是因为**kibana.yml**配置了：

```yaml
# 汉化
i18n.locale: "zh-CN"
```

![](kibana.jpg)



## IK分词器

IK分词器有两种分词形式：

* ik_smart：最少切分，切分以只要词进行划分
* ik_max_word：最细粒度划分，会根据所有可能存在的词进行划分

**ik_smart**

请求：_analyze是分词请求

```json
GET _analyze
{
  "analyzer": "ik_smart",
  "text": "我是中国人"
}
```

响应：

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "中国人",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    }
  ]
}
```

**ik_max_word**

请求：

```json
GET _analyze
{
  "analyzer": "ik_max_word",
  "text": "我是中国人"
}
```

响应：

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "中国人",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "中国",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "国人",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 4
    }
  ]
}
```

#### 自定义词典

例如：我们对“我是夜月蜩”进行分词，两种分词的计算结果都是以下结果

![](我是夜月蜩.jpg)

字典里并没有“夜月蜩”，所以分词并不认为其是一个词，这时候我们可以自定义分词

在ik分词器的config目录下有一个IKAnalyzer.cfg.xml

![](自定义分词.jpg)

我们可以在文件里定义自己的分词字典

![](myself分词.jpg)

然后在ik分词器的config目录下创建名为myself.dic的自定义词典，词典里写入“夜月蜩”

![](分词结果.jpg)

重启ES，进行测试.这时候ik_smart和ik_max_word都会把“夜月蜩”当成一个词

![](夜月蜩结果.jpg)

## Elasticsearch的使用

#### Rest风格

基于elasticsearch的Rest风格说明如下表：

| 序号 | method | Url地址                                         | 描述                   |
| ---- | ------ | ----------------------------------------------- | ---------------------- |
| 1    | PUT    | localhost:9002/索引名称/类型名称/文档id         | 创建文档（指定文档id） |
| 2    | POST   | localhost:9002/索引名称/文档id                  | 创建文档（随机文档id） |
| 3    | POST   | localhost:9002/索引名称/类型名称/文档id/_update | 修改文档               |
| 4    | DELETE | localhost:9002/索引名称/类型名称/文档id         | 删除文档               |
| 5    | GET    | localhost:9002/索引名称/类型名称/文档id         | 查询文档（通过文档id） |
| 6    | POST   | localhost:9002/索引名称/类型名称/_search        | 查询所有数据           |

序号1的PUT也能修改文档，只要修改了文档里的数据后再次执行一遍，就会把最初创建的数据给替换掉，但是这个要求执行时以前有的数据需要都在，否则就会造成数据为空

而序号3的POST修改文档，可以只指定某一条需要修改的数据进行修改

#### Elasticsearch的数据类型

字符串类型：

* type：可分割
* keyword：不可分割，如果指定的是keyword，那查询时严格按照原始值查询

数值类型

* long，integer，short，byte，double，float，half float，scaled float

日期类型

* date

 布尔值类型

* boolean

二进制类型

* binary

#### Elasticsearch的增删改查

**建索引字段信息**

```json
PUT /test1
{
  "mappings": {
    "properties": {
      "name":{
        "type": "text"
      },
      "age":{
        "type": "integer"
      },
      "date":{
        "type": "date"
      }
    }
  }
}
```

这样创建的索引只有字段，字段里没有值

**获取索引字段信息**

```json
GET /test1
```

**创建文档**

```json
PUT /test/user/1
{
  "name":"zhangsan",
  "age":12,
  "desc":"历史第一渣男",
  "date":"2020-08-31"
}
```

**修改文档1**

直接在上面创建文档的基础上修改name=“lisi”，但是这样修改里面的age，desc，date属性都要在，这种修改相当于文档里所有属性的修改，如果有属性没有设置，则修改好的文档里也没有该属性

```json
PUT /test/user/1
{
  "name":"lisi",
  "age":12,
  "desc":"历史第一渣男",
  "date":"2020-08-31"
}
```

**修改文档2**

这种修改方式可以只修改制定的属性

```json
POST /test/user/1/_update
{
  "doc": {
    "name":"wangwu"
  }
}
```

**删除文档**

```json
DELETE /test/user/1
```

**查询类型下所有文档**

```json
GET /test/user/_search
```

**附带条件查询**

一般不采用这种条件查询方式

```json
GET /test/user/_search?q=desc:"男"
```

#### Elasticsearch的复杂查询

包括：排序，分页，高亮，模糊查询，精准查询

**条件查询**

查询匹配name中带有“李”字的

```json
GET /test/user/_search
{
  "query": {
    "match": {
      "name": "李"
    }
  }
}
```

查询结果：

```json
#! Deprecation: [types removal] Specifying types in search requests is deprecated.
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.4991763,
    "hits" : [
      {
        "_index" : "test",
        "_type" : "user",
        "_id" : "2",
        "_score" : 0.4991763,
        "_source" : {
          "name" : "李三",
          "age" : 13,
          "desc" : "历史第一美男",
          "date" : "2020-08-31"
        }
      },
      {
        "_index" : "test",
        "_type" : "user",
        "_id" : "3",
        "_score" : 0.4208172,
        "_source" : {
          "name" : "小李子",
          "age" : 13,
          "desc" : "历史第一花女",
          "date" : "2020-08-31"
        }
      }
    ]
  }
}
```

**按条件查询出指定属性**

因为所有的属性都在_source下，所以要查询出只包含name和desc属性的数据，就直接指定就行

```json
GET /test/user/_search
{
  "query": {
    "match": {
      "name": "李"
    }
  },
  "_source": ["name","desc"]
}
```

**排序查询**

"order": "desc"  表示降序排序

"order": "asc"     表示升序排序

如果指定了排序规则，则_score就为null

```json
GET /test/user/_search
{
  "query": {
    "match": {
      "name": "李"
    }
  },
  "sort": [
    {
      "age": {
        "order": "desc"
      }
    }
  ]
}
```

**分页查询**

"from": 0  表示从第1个数据开始，第一个数据的下标是0

"size": 1   表示每一页显示1个

```json
GET /test/user/_search
{
  "from": 0,
  "size": 1
}
```

**布尔值查询**

作用：可以进行多条件查询

> 查询出：name中带有“三”字，并且age=14的文档

这里的must相当于：and

 ```json
GET /test/user/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "三"
          }
        },
        {
          "match": {
            "age": "14"
          }
        }
      ]
    }
  }
}
 ```

> 查询出：name中带有“三”字，或者age=13的文档

这里的should相当于or

```json
GET /test/user/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "name": "三"
          }
        },
        {
          "match": {
            "age": "13"
          }
        }
      ]
    }
  }
}
```

> 查询出：name中不带“三”字的文档

这里的must_not相当于not

```json
GET /test/user/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "match": {
            "name": "三"
          }
        }
      ]
    }
  }
}
```

**过滤查询**

>  查询出name中有“三”字的，并且年龄大于等于14岁的

filter:表示过滤，这里按照age的range区间进行过滤

```json
GET /test/user/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "三"
          }
        }
      ],
      "filter": {
        "range": {
          "age": {
            "gte": 14
          }
        }
      }
    }
  }
}
```

gte：大于等于

gt：大于

lte：小于等于

lt：小于

这些符号可以联合使用

```json
GET /test/user/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "三"
          }
        }
      ],
      "filter": {
        "range": {
          "age": {
            "gte": 10, 
            "lt": 14
          }
        }
      }
    }
  }
}
```

**模糊查询**

“历史”和“男”中间用空格隔开，这个会把文档中”desc“属性只要有”历史“或者”男“的文档都会被查出来，只是根据配置程度_score的值会不同

```json
GET /test/user/_search
{
  "query": {
    "match": {
      "desc": "历史 男"
    }
  }
}
```

**精确查询**

使用term进行精确查询

```json
GET /test/user/_search
{
  "query": {
    "term": {
      "name": "三"
    }
  }
}
```

这样的查询仍然会查询出name中所有包含“三”字的文档，这是因为name的字段类型是text类型，必须在建立索引字段类型时是keyword类型才能精确查询

```json
PUT /test2
{
  "mappings": {
    "properties": {
      "name":{
        "type": "keyword"
      },
      "age":{
        "type": "integer"
      }
    }
  }
}

PUT /test2/_doc/1
{
  "name":"小王java",
  "age":12
}

PUT /test2/_doc/2
{
  "name":"小王",
  "age":12
}
```

这样执行查询时就能精确查询了

只会查询出name=小王的文档

```json
GET /test2/_search
{
  "query": {
    "term": {
      "name": "小王"
    }
  }
}
```

**高亮查询**

查询出所有name名字中带“三”的文档，并对"三"字高亮

```json
GET /test/user/_search
{
  "query": {
    "match": {
      "name": "三"
    }
  },
  "highlight": {
    "fields": {
      "name":{}
    }
  }
}
```

**自定义高亮样式查询**

pre_tags：高亮字之前的部分

post_tags：高亮字之后的部分

这样处理的name结果："李<p class='key' style='color:red'>三</p>"

```json
GET /test/user/_search
{
  "query": {
    "match": {
      "name": "三"
    }
  },
  "highlight": {
    "pre_tags": "<p class='key' style='color:red'>", 
    "post_tags": "</p>",
    "fields": {
      "name":{}
    }
  }
}
```

## Springboot整合Eslasticsearch

[官方使用文档](http://jvm123.com/doc/es/index.html#elasticsearch.operations.template)

我这里使用的Springboot是2.2.5.RELEASE版本，elasticsearch是7.6.1版本，如果Springboot版本太低，是没有RestHighLevelClient这个类的

**pom依赖**

```xml
<properties>
    <java.version>1.8</java.version>
    <elasticsearch.version>7.6.1</elasticsearch.version>
</properties>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
<!--操作数据时需要json的格式-->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.58.sec06</version>
</dependency>
```

**application.properties配置**

```properties
# 不做任何配置
```

**配置类**

com.hui.config.ElasticsearchConfig

```java
package com.hui.config;

import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestHighLevelClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ElasticsearchConfig {

    @Bean(destroyMethod = "close")
    public RestHighLevelClient restHighLevelClient(){
        RestHighLevelClient client=new RestHighLevelClient(
                RestClient.builder(
                        new HttpHost("192.168.31.181", 9200, "http")));
        return client;
    }
}
```

#### index测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootesApplicationTests {
    
    @Autowired
    @Qualifier("restHighLevelClient")
    private RestHighLevelClient client;

    /**
     * 创建索引，type默认为_doc
     * @throws IOException
     */
    @Test
    public void createIndex() throws IOException {
        CreateIndexRequest request=new CreateIndexRequest("index0");
        CreateIndexResponse response = client.indices().create(request,RequestOptions.DEFAULT);
        System.out.println(response);

    }

    /**
     * 获取索引，判断索引是否存在
     * @throws IOException
     */
    @Test
    public void getIndex() throws IOException {
        GetIndexRequest request=new GetIndexRequest("index0");
        //获取索引
        GetIndexResponse response = client.indices().get(request, RequestOptions.DEFAULT);
        System.out.println(response);
        //判断索引是否存在
        boolean isExists = client.indices().exists(request, RequestOptions.DEFAULT);
        System.out.println(isExists);
    }

    /**
     * 删除索引
     * @throws IOException
     */
    @Test
    public void deleteIndex() throws IOException {
        DeleteIndexRequest request=new DeleteIndexRequest("index0");
        //删除索引
        AcknowledgedResponse response = client.indices().delete(request, RequestOptions.DEFAULT);
        //判断是否删除成功，为true为删除
        System.out.println(response.isAcknowledged());

    }
}
```

#### 文档测试

**实体类**

```java
package com.hui.pojo;

public class User {

    private String name;
    private Integer age;

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

**创建文档**

```java
/**
     * 创建文档，type默认为_doc
     * @throws IOException
     */
@Test
public void addDocument() throws IOException {
    User user=new User("张三",13);
    //创建请求
    IndexRequest request=new IndexRequest("index1");

    //设置索引文档下的id
    request.id("1");
    //设置请求超时时间为1s
    request.timeout(TimeValue.timeValueSeconds(1));
    //设置_source里的数据
    request.source(JSON.toJSONString(user), XContentType.JSON);

    IndexResponse response = client.index(request, RequestOptions.DEFAULT);
    //创建文档返回的相信信息
    System.out.println(response.toString());
    //创建是否成功的状态 CREATED
    System.out.println(response.status());
}
```

**获取指定id文档**

```java
/**
     * 获取文档信息
     * @throws IOException
     */
@Test
public void getDocument() throws IOException {
    //获取请求
    GetRequest request=new GetRequest("index1","1");

    GetResponse response = client.get(request, RequestOptions.DEFAULT);
    //判断查询结果是否存在
    System.out.println(response.isExists());
    //这个获取的就是GET /index1/_doc/1 所查询的信息
    System.out.println(response.toString());
    //这个只获取_source下的信息
    System.out.println(response.getSourceAsString());
}
```

**更新指定id文档**

```java
/**
     * 更新文档信息
     * @throws IOException
     */
@Test
public void updateDocument() throws IOException {
    //更新的请求
    UpdateRequest request=new UpdateRequest("index1","1");

    //设置请求超时时间为1s
    request.timeout(TimeValue.timeValueSeconds(1));

    User user=new User("王五",14);
    request.doc(JSON.toJSONString(user), XContentType.JSON);

    UpdateResponse response = client.update(request, RequestOptions.DEFAULT);

    //获取更新后的结果
    System.out.println(response.toString());
    //获取更新状态 OK
    System.out.println(response.status());
}
```

**删除指定id文档**

```java
/**
     * 删除文档信息
     * @throws IOException
     */
@Test
public void deleteDocument() throws IOException {
    //删除的请求
    DeleteRequest request=new DeleteRequest("index1","1");

    //设置请求超时时间为1s
    request.timeout(TimeValue.timeValueSeconds(1));

    DeleteResponse response = client.delete(request, RequestOptions.DEFAULT);

    //获取删除后的结果
    System.out.println(response.toString());
    //获取删除更新状态 OK
    System.out.println(response.status());
}
```

**批量插入文档**

使用的是：BulkRequest

```java
/**
     * 批量插入文档
     * @throws IOException
     */
@Test
public void addBulkDocument() throws IOException {
    BulkRequest bulkrequest=new BulkRequest();
    bulkrequest.timeout(TimeValue.timeValueSeconds(1));

    List<User> lists=new ArrayList<>();
    lists.add(new User("王一",11));
    lists.add(new User("王二",12));
    lists.add(new User("王三",13));
    lists.add(new User("王四",14));
    lists.add(new User("王五",15));
    lists.add(new User("王六",16));

    for (int i=0;i<lists.size();i++){
        bulkrequest.add(
            //插入的索引
            new IndexRequest("index2")
            .id(String.valueOf(i+1))    //插入的id
            .source(JSON.toJSONString(lists.get(i)),XContentType.JSON)  //插入的文档
        );
    }

    BulkResponse bulk = client.bulk(bulkrequest, RequestOptions.DEFAULT);
    //OK
    System.out.println(bulk.status());
}
```

**复杂查询**

使用的是：SearchRequest

```java
/**
     * 复杂查询
     * SearchRequest：构建搜索请求
     * SearchSourceBuilder：构建搜索条件对象
     * MatchQueryBuilder：构建搜索匹配规则
     * MatchAllQueryBuilder：查询所有规则
     * TermQueryBuilder：精确匹配规则
     * @throws IOException
     */
@Test
public void complexSearch() throws IOException {
    //构建搜索请求
    SearchRequest searchRequest=new SearchRequest("index2");
    //构建搜索条件
    SearchSourceBuilder builder=new SearchSourceBuilder();

    //MatchAllQueryBuilder matchAllQueryBuilder = QueryBuilders.matchAllQuery();//查询所有，相当于GET /index2/_doc/_search
    //匹配查询，其中QueryBuilders.termQuery()是精确查询
    MatchQueryBuilder matchQueryBuilder = QueryBuilders.matchQuery("name", "王");
    //设置高亮
    builder.highlighter();
    //设置查询条件
    builder.query(matchQueryBuilder);
    builder.timeout(TimeValue.timeValueSeconds(1));

    //将搜索条件放入到查询请求中
    searchRequest.source(builder);

    //执行查询请求
    SearchResponse search = client.search(searchRequest, RequestOptions.DEFAULT);
    //所有查询到的内容全部在hits中
    SearchHits hits = search.getHits();
    //获取一共多少条文档
    System.out.println(hits.getTotalHits());
    for (SearchHit searchHit:hits.getHits()){
        //获取每个文档下的_source内容
        System.out.println(searchHit.getSourceAsString());
    }

}
```

