---
title: ElasticSearch基础
date: 2017-11-23 14:09:33
tags:
    - ElasticSearch
categories: 工具
---

## 增删查改范例

>  Restful vs Java API

### Java API 说明

需要添加依赖：

```xml
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>transport</artifactId>
            <version>5.6.4</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.9.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.9.1</version>
        </dependency>
```

需要获取 client 实例:

```Java
        Settings settings = Settings.EMPTY;
        TransportClient client = new PreBuiltTransportClient(settings)
                .addTransportAddress(new InetSocketTransportAddress(InetAddress.getLocalHost(), 9300));
```

### 增

#### Restful

`curl -XPOST 'localhost:9200/books/es/1' -d '{"title": "Elasticsearch Server", "published": 2013}'`

返回：

`{"_index":"books","_type":"es","_id":"1","_version":1,"result":"created","_shards":{"total":2,"successful":1,"failed":0},"created":true}`

<!-- more -->

#### Java API

```Java
        // add，可以直接使用 json 添加，也可以像下面一样使用
        XContentBuilder builder = XContentFactory.jsonBuilder().startObject()
                .field("title", "Mastering Elasticsearch").field("published", "2013").endObject();
        String json = builder.string();
        System.out.println(json);
        // IndexResponse indexResponse = client.prepareIndex("books", "es", "2")
        // .setSource(json, XContentType.JSON).get();
        IndexResponse indexResponse = client.prepareIndex("books", "es", "2")
                .setSource(builder).get();
        System.out.println(indexResponse);


/**
结果:

{"title":"Mastering Elasticsearch","published":"2013"} // JSON
IndexResponse[index=books,type=es,id=2,version=1,result=created,shards={"total":2,"successful":1,"failed":0}] // indexResponse
**/
```

### 查

#### Restful

`curl -XGET 'localhost:9200/books/_search?pretty'`

```json
{
  "took" : 41,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 3,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "books",
        "_type" : "es",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "title" : "Mastering Elasticsearch",
          "published" : 2013
        }
      },
      {
        "_index" : "books",
        "_type" : "es",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "title" : "Elasticsearch Server",
          "published" : 2013
        }
      },
      {
        "_index" : "books",
        "_type" : "solr",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "title" : "Apache Solr 4 Cookbook",
          "published" : 2012
        }
      }
    ]
  }
}
```

`curl -XGET 'localhost:9200/books/_search?pretty&q=title:elasticsearch'`

```json
{
  "took" : 12,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 0.7373906,
    "hits" : [
      {
        "_index" : "books",
        "_type" : "es",
        "_id" : "1",
        "_score" : 0.7373906,
        "_source" : {
          "title" : "Elasticsearch Server",
          "published" : 2013
        }
      },
      {
        "_index" : "books",
        "_type" : "es",
        "_id" : "2",
        "_score" : 0.25811607,
        "_source" : {
          "title" : "Mastering Elasticsearch",
          "published" : 2013
        }
      }
    ]
  }
}
```

#### Java API

##### 通过 id 查询

```Java
        // get by id
        GetResponse response = client.prepareGet("books", "es", "2").get();
        System.out.println(response);

/**
结果:

{"_index":"books","_type":"es","_id":"2","_version":1,"found":true,"_source":{"title":"Mastering Elasticsearch","published":"2013"}}
**/
```

##### 通过 query 查询

```Java
        SearchResponse response = client.prepareSearch("books")
                .setQuery(
                        QueryBuilders.termsQuery("title","elasticsearch")
                )
                .get();
        System.out.println(response);

/**
结果

{"took":11,"timed_out":false,"_shards":{"total":5,"successful":5,"skipped":0,"failed":0},"hits":{"total":2,"max_score":0.7373906,"hits":[{"_index":"books","_type":"es","_id":"1","_score":0.7373906,"_source":{"title": "Elasticsearch Server", "published": 2013}},{"_index":"books","_type":"es","_id":"2","_score":0.25811607,"_source":{"title":"Mastering Elasticsearch","published":"2013"}}]}}
**/
```

### 改

#### Restful

更新索引中的文档是一项复杂的任务。在内部，ElasticSearch 必须首先后去文档，从 `_source` 属性获得数据，删除旧的文件，更改 `_source` 属性，然后把它作为新的文档来索引。它如此复杂，因为信息一旦在 Lucene 的倒排索引中存储，就不能再被更改。

`curl -XPOST 'localhost:9200/books/es/2/_update' -d '{"script": "ctx._source.published = \"2017\"" }'`

`{"_index":"books","_type":"es","_id":"2","_version":2,"result":"updated","_shards":{"total":2,"successful":1,"failed":0}}`

结果验证:

`curl -XGET 'localhost:9200/books/_search?pretty&q=title:mastering'`

```Java
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 0.25811607,
    "hits" : [
      {
        "_index" : "books",
        "_type" : "es",
        "_id" : "2",
        "_score" : 0.25811607,
        "_source" : {
          "title" : "Mastering Elasticsearch",
          "published" : "2017"
        }
      }
    ]
  }
}
```

#### Java API

```Java
        // use prepareUpdate
        UpdateResponse response = client.prepareUpdate("books", "es", "2").setScript(
                new Script("ctx._source.published = \"2017\"")
        ).get();
        UpdateResponse response = client.prepareUpdate("books", "es", "2")
                .setDoc(
                        XContentFactory.jsonBuilder().startObject().field("published", "2017").endObject()
                )
                .get();

        // use UpdateRequest
        UpdateResponse response = client.update(
                new UpdateRequest("books", "es", "2")
                        .doc(
                                XContentFactory.jsonBuilder().startObject().field("published", "2017").endObject()
                        )
        ).get();
        System.out.println(response);

/**
结果：

UpdateResponse[index=books,type=es,id=2,version=2,result=updated,shards=ShardInfo{total=2, successful=1, failures=[]}]
**/
```

### 删

#### Restful

`curl -XDELETE 'localhost:9200/books/es/2'`

`{"found":true,"_index":"books","_type":"es","_id":"2","_version":3,"result":"deleted","_shards":{"total":2,"successful":1,"failed":0}}`

结果验证:

` curl -XGET 'localhost:9200/books/_search?pretty&q=title:mastering'`

```Java
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 0,
    "max_score" : null,
    "hits" : [ ]
  }
}
```

#### Java API

```Java
        DeleteResponse response = client.prepareDelete("books", "es", "2").get();
        System.out.println(response);

/**
结果:

DeleteResponse[index=books,type=es,id=2,version=5,result=deleted,shards=ShardInfo{total=2, successful=1, failures=[]}]
**/
```

## 基础概念

### 数据架构的主要概念

- 索引

  索引（index） 是 Elasticsearch 对逻辑数据的逻辑存储，所以它可以分为更小的部分。你可以**把索引看成关系型数据库的表**。然而，索引的结构是为快速有效的全文索引准备的，特别是**它不存储原始值**。Elasticsearch 可以把索引存放在一台机器或者分散在多台服务器上，**每个索引有一或多个分片（shard），每个分片可以有多个副本（replica）。**

- 文档

  存储在 Elasticsearch 中的**主要实体**叫文档（ document）。用关系型数据库来类比的话，**一个文档相当于数据库表中的一行记录**。

  文档由多个字段组成，每个字段可能多次出现在一个文档里，这样的字段叫**多值字段**（multivalued）。每个字段有类型，如文本、数值、日期等。字段类型也可以是复杂类型，一个字段包含其他子文档或者数组。

  **每个文档存储在一个索引中并有一个 Elasticsearch 自动生成的唯一标识符和文档类型**。文档需要有对应文档类型的唯一标识符，这意味着在一个索引中，**两个不同类型的文档可以有相同的唯一标识符**。

- 文档类型

  在 Elasticsearch 中，一个索引对象可以存储很多不同用途的对象。例如，一个博客应用程序可以保存文章和评论。文档类型让我们轻易地区分单个索引中的不同对象。每个文档可以有不同的结构，但在实际部署中，将文件按类型区分对数据操作有很大帮助。当然，需要记住一个限制，**不同的文档类型不能为相同的属性设置 不同的类型。例如，在同一索引中的所有文档类型中，一个叫 title 的字段必须具有相同的类型。**

- 映射

  **文档中的每个字段都必须根据不同类型做相应的分析。Elasticsearch 在映射中存储有关字段的信息**。每一个 文档类型都有自己的映射，即使我们没有明确定义。

### 主要概念

- 节点和集群

  Elasticsearch 可以运行在许多互相合作的服务器上。这些服务器称为集群（cluster），形成集群的每个服务器称为节点（node）。

- 分片

  当有大量的文档时，由于内存的限制、硬盘能力、处理能力不足、**无法足够快地响应客户端请求等**，**一个节点可能不够。在这种情况下，数据可以分为较小的称为分片（shard）的部分**（其中**每个分片都是一个独立的ApacheLucene 索引**）。每个分片可以放在不同的服务器上，因此，数据可以在集群的节点中传播。**当你查询的索引分布在多个分片上时，Elasticsearch 会把查询发送给每个相关的分片，并将结果合并在一起，而应用程序并不知道分片的存在。此外，多个分片可以加快索引。**

- 副本

  为了**提高查询吞吐量或实现高可用性**，可以使用分片副本。**副本（replica）只是一个分片的精确复制**，每个分片可以有零个或多个副本。换句话说，**Elasticsearch 可以有许多相同的分片，其中之一被自动选择去更改索引操作。这种特殊的分片称为主分片（primaryshard），其余称为副本分片（replicashard）**。在主分片丢失时，例如该分片数据所在服务器不可用，集群将副本提升为新的主分片。

  ​

## 基本查询

#### 词条查询（`term`）

词条查询是 Elasticsearch 中的一个简单查询。**它仅匹配在给定字段中含有该词条的文档，而且是确切的、未经分析的词条**。

```shell
                                    
     {
       "query" : {
          "term" : {
             "title": "elasticsearch" 
          }
       }
     }'
     
# 另外的查询 json
          {
            "query" : {
               "term" : {
                  "title": {
                       "value": "elasticsearch",
                       "boost": 10.0
                  }
               }
            }
          }'
```

由于索引内容全改为小写，所以搜索的也改为小写

#### 多词条查询（`terms-tags`）

```json
          {
            "query" : {
               "terms" : {
                  "published" : [ "2013", "2012" ]
               }
            }
          }
```

#### 匹配所有文件（`match_all`）

```json
          {
            "query" : {
               "match_all": {}
            }
          }
```

#### 常用词（`common`）

常用词查询是在没有使用停用词（stopword，http://en.wikipedia.org/wiki/Stop_words）的情况下，Elasticsearch为了提高常用词的查询相关性和精确性而提供的一个现代解决方案。例如，“book and coffee”可以翻译成3个词查询，每一个都有性能上的成本（词越多，查询性能越低）。**但『and』这个词非常常见，对文档得分的影响非常低。解决办法是常用词查询，将查询分为两组。第一组包含重要的词，出现的频率较低。第二组包含较高频率的、不那么重要的词。**先执行第一个查询，Elasticsearch从第一组的所有词中计算分数。这样，通常都很重要的低频词总是被列入考虑范围。然后，Elasticsearch对第二组中的词执行二次查询，但只为与第一个查询中匹配的文档计算得分。这样只计算了相关文档的得分，实现了更高的性能。

https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-common-terms-query.html

```json
{
    "query": {
        "common": {
            "title": {
                "query": "this is bonsai cool",
                "cutoff_frequency": 0.001
            }
        }
    }
}
```

查询可以使用下列的参数：

- query：定义实际查询内容
- cutoff_frequency：这个参数定义一个百分比（0.001表示0.1%）或一个绝对值（当此属性值>=1时）。**这个值用来构建高、低频词组。此参数设置为0.001意味着频率<=0.1%的词将出现在低频词组中。**
- low_freq_operator：这个参数可以设为 or 或 and，默认是or。它用来**指定为低频词组构建查询时用到的布尔运算符**。如果希望所有的词都在文档中出现才认为是匹配，应该把它设置为and。
- high_freq_operator：这个参数可以设为 or 或a nd，默认是or。它用来**指定为高频词组构建查询时用到的布尔运算符**。如果希望所有的词都在文档中出现才认为是匹配，那么应该把它设置为and。
- minimum_should_match：不使用 low_freq_operator 和 high_freq_operator 参数的话，可以使用使用minimum_should_match 参数。和其他查询一样，它**允许指定匹配的文档中应该出现的查询词的最小个数**。
- boost：这个参数定义了赋给文档得分的**加权值**。
- analyzer：这个参数**定义了分析查询文本时用到的分析器名称**。默认值为 default analyzer。
- disable_coord：此参数的值默认为 false，它**允许启用或禁用分数因子的计算，该计算基于文档中包含的所有查询词的分数**。把它设置为true，得分不那么精确，但查询将稍快。

#### match 查询（`match`）

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html)

match 查询把 query 参数中的值拿出来，加以分析，然后构建相应的查询**。使用 match 查询时，Elasticsearch 将对一个字段选择合适的分析器，所以可以确定，传给 match 查询的词条将被建立索引时相同的分析器处理**。请记住，**match 查询（以及将在稍后解释的 multi_match 查询）不支持 Lucene 查询语法**。但是，它是完全符合搜索需求的一个查询处理器。

最简单的查询

```json
{
    "query": {
        "match" : {
            "message" : "this is a test"
        }
    }
}
```

下面我们看看它几种类型

##### 布尔值

```Json
{
    "query": {
        "match" : {
            "message" : {
                "query": "this is a test",
              	"operator": "and"
            }
        }
    }
}
```



布尔匹配查询分析提供的文本，然后做出布尔查询。有几个参数允许控制布尔查询匹配行为：

- operator：此参数可以接受 or 和 and，**控制用来连接创建的布尔条件的布尔运算符**。默认值是or。如果希望查询中的所有条件都匹配，可以使用and运算符。
- analyzer：这个参数定义了分析查询文本时用到的分析器的名字。默认值为 default analyzer。
- fuzziness：**可以通过提供此参数的值来构建模糊查询**（fuzzy query）。它为字符串类型提供从0.0到1.0的值。构造模糊查询时，该参数将用来设置相似性。
- prefix_length：**此参数可以控制模糊查询的行为**。
- max_expansions：**此参数可以控制模糊查询的行为**。
- zero_terms_query：该参数允许**指定当所有的词条都被分析器移除时（例如，因为停止词），查询的行为。它可以被设置为none或all，默认值是none。**在分析器移除所有查询词条时，该参数设置为none，将没有文档返回；设置为all，则将返回所有文档。
- cutoff_frequency：**该参数允许将查询分解成两组：一组低频词和一组高频词**。

##### match_phrase

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html)

```Json
{
    "query": {
        "match_phrase" : {
            "message" : {
                "query" : "this is a test",
                "analyzer" : "my_analyzer"
            }
        }
    }
}
```

match_phrase 查询类似于布尔查询，不同的是，**它从分析后的文本中构建短语查询，而不是布尔子句**。该查询可以使用下面几种参数：

- slop：这是一个整数值，**该值定义了文本查询中的词条和词条之间可以有多少个未知词**条，以被视为跟一个短语匹配。此参数的默认值是0，这意味着，不允许有额外的词条1。
- analyzer：这个参数定义定义了分析查询文本时用到的分析器的名字。默认值为 default analyzer。

##### match_phrase_prefix

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase-prefix.html)

```Java
{
    "query": {
        "match_phrase_prefix" : {
            "message" : {
                "query" : "quick brown f",
                "max_expansions" : 10
            }
        }
    }
}
```

match_query 查询的最后一种类型是 match_phrase_prefix 查询。此查询跟 match_phrase 查询几乎一样，但除此之外，**它允许查询文本的最后一个词条只做前缀匹配**。此外，除了 match_phrase 查询公开的参数，**还公开了一个额外参数max_expansions。这个参数控制有多少前缀将被重写成最后的词条。**

#### multi_match

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html)

可以通过 fields 指定多字段进行 match 查询

```json
{
  "query": {
    "multi_match" : {
      "query":    "this is a test", 
      "fields": [ "subject", "message" ] 
    }
  }
}
```

除了 match 提供的参数，它还可以通过以下参数来控制行为:

- use_dis_max：该参数定义一个布尔值，**设置为 true 时，使用析取最大分查询，设置为false时，使用布尔查询**。默认值为 true。
- tie_breaker：只有在 use_dis_max 参数设为 true 时才会使用这个参数。**它指定低分数项和最高分数项之间的平衡**。

#### query_string 查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html)

相比其他查询， query_string 支持全部 Apache Lucene 查询语法

```json
{
    "query": {
        "query_string" : {
            "query" : "city.\\*:(this AND that OR thus)"
        }
    }
}
```

##### 针对多字段

将 `use_ids_max` 设置为 true

#### simple_query_string 查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-simple-query-string-query.html)

使用 Lucene 最新查询解析器之一: `SimpleQueryParser`

```json
{
  "query": {
    "simple_query_string" : {
        "query": "\"fried eggs\" +(eggplant | potato) -frittata",
        "fields": ["title^5", "body"],
        "default_operator": "and"
    }
  }
}
```

#### 标识符查询

仅用于提供的标识符来过滤返回的文档，针对内部 _uid 字段运行

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-ids-query.html)

```json
{
    "query": {
        "ids" : {
            "type" : "my_type",
            "values" : ["1", "4", "100"]
        }
    }
}
```

#### 前缀查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-prefix-query.html)

```json
{ 
  "query": {
    "prefix" : { "user" : "ki" }
  }
}
```

#### fuzzy 查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html)

基于编辑距离算法来匹配文档。编辑距离的计算基于我们提供的查询词条和被搜索文档。此查询很占用 CPU 资源，但当**需要模糊匹配时它很有用**，例如，当用户拼写错误时。

```json
{
    "query": {
        "fuzzy" : {
            "user" : {
                "value" :         "ki",
                    "boost" :         1.0,
                    "fuzziness" :     2,
                    "prefix_length" : 0,
                    "max_expansions": 100
            }
        }
    }
}
```

可以使用下面的参数来控制 fuzzy 查询的行为。

- value：此参数指定了实际的查询。
- boost：此参数指定了查询的加权值，默认为1.0。
- min_similarity：**此参数指定了一个词条被算作匹配所必须拥有的最小相似度**。对字符串字段来说，这个值应该在0到1之间，包含0和1。对于数值型字段，这个值可以大于1，比如查询值是20，min_similarity设为3，则可以得到17~23的值。对于日期字段，可以把min_similarity参数值设为1d、2d、1m等，分别表示1天、2天、1个月。
- prefix_length：此参数**指定差分词条的公共前缀长度**，默认值为0。
- max_expansions：此参数**指定查询可被扩展到的最大词条数**，默认值是无限制。

#### 通配符查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-wildcard-query.html)

通配符查询允许我们在查询值中使用 * 和 ? 等通配符。此外，通配符查询跟词条查询在内容方面非常类似。

```Json
{
    "query": {
        "wildcard" : { "user" : { "value" : "ki*y", "boost" : 2.0 } }
    }
}
```

#### more_like_this 查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-mlt-query.html)

more_like_this 查询让我们能够得到与提供的文本类似的文档。Elasticsearch 支持几个参数来定义 more_like_this 查询如何工作，如下所示。

```Json
{
    "query": {
        "more_like_this" : {
            "fields" : ["title", "description"],
            "like" : "Once upon a time",
            "min_term_freq" : 1,
            "max_query_terms" : 12
        }
    }
}
```

- fields：此参数定义**应该执行查询的字段数组**，默认值是 _all 字段。
- like_text：这是一个必需的参数，包含**用来跟文档比较的文本**。
- percent_terms_to_match：此参数**定义了文档需要有多少百分比的词条与查询匹配才能认为是类似**的，默认值为0.3，意思是30%。
- min_term_freq：此参数**定义了文档中词条的最低词频**，低于此频率的词条将被忽略，默认值为2。
- max_query_terms：此参数**指定生成的查询中能包括的最大查询词条数**，默认值为25。值越大，精度越大，但性能也越低。
- stop_words：此参数**定义了一个单词的数组，当比较文档和查询时，这些单词将被忽略**，默认值为空数组。
- min_doc_freq：此参数**定义了包含某词条的文档的最小数目**，低于此数目时，该词条将被忽略，默认值为5，意味着一个词条至少应该出现在5个文档中，才不会被忽略。
- max_doc_freq：此参数**定义了包含某词条的文档的最大数目**，高于此数目时，该词条将被忽略，默认值为无限制。
- min_word_len：此参数**定义了单词的最小长度**，低于此长度的单词将被忽略，默认值为0。
- max_word_len：此参数**定义了单词的最大长度**，高于此长度的单词将被忽略，默认值为无限制。
- boost_terms：此参数定义了用于每个**词条的加权值**，默认值为1。
- boost：此参数定义了用于**查询的加权值**，默认值为1。
- analyzer：此参数指定了针对我们提供的**文本的分析器名称**。

#### 范围查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-range-query.html)

```json
{
    "query": {
        "range" : {
            "age" : {
                "gte" : 10,
                "lte" : 20,
                "boost" : 2.0
            }
        }
    }
}
```

- gte：范围查询将匹配字段值大于或等于此参数值的文档。
- gt：范围查询将匹配字段值大于此参数值的文档。
- lte：范围查询将匹配字段值小于或等于此参数值的文档。
- lt：范围查询将匹配字段值小于此参数值的文档。

#### 正则表达式查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-regexp-query.html)

```json
{
    "query": {
        "regexp":{
            "name.first": "s.*y"
        }
    }
}
```

## 版本控制

之前查询返回的结果中有一个 `"_version": 1`

仔细观察，你会发现在更新相同标识符的文档后，这个版本是递增的。默认情况下，Elasticsearch在添加、更改或删除文档时都会递增版本号。除了告诉我们对文档所做更改的次数，还能够实现**[乐观锁](https://zh.wikipedia.org/wiki/%E4%B9%90%E8%A7%82%E5%B9%B6%E5%8F%91%E6%8E%A7%E5%88%B6)**。
