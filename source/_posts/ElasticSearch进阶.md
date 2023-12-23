---
title: ElasticSearch进阶
date: 2017-12-06 10:22:12
tags:
    - ElasticSearch
categories: 工具
---

## 索引

### 映射

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/mapping.html)

简而言之，映射即为结构（虽然说 ElasticSearch 是一个无模式的搜索引擎）。

定义方式：

```json
curl -XPUT 'localhost:9200/my_index?pretty' -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "doc": { 
      "properties": {  // 字段定义
        "title":    { "type": "text"  }, 
        "name":     { "type": "text"  }, 
        "age":      { "type": "integer" },  
        "created":  {
          "type":   "date",   // 类型定义
          "format": "strict_date_optional_time||epoch_millis"
        }
      }
    }
  }
}
'
```

<!-- more -->

#### 核心类型

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/mapping-types.html)

每个字段类型可以指定为 Elasticsearch 提供的一个特定核心类型。Elasticsearch 有以下核心类型：

- string：字符串。`text` 与 `keyword`
- number: 数字。`long`, `integer`, `short`, `byte`, `double`, `float`, `half_float`, `scaled_float`
- date：日期。`date`
- boolean：布尔型。`boolean`
- binary：二进制。`binary`
- range: 范围值。`integer_range`, `float_range`, `long_range`, `double_range`, `date_range`

除了核心类型之外还有复杂的数组，对象，内嵌对象类型，地理位置，特有类型。

##### 公共属性

- index_name：该属性定义将存储在索引中的字段名称。若未定义，字段将以对象的名字来命名。
- index：可设置值为 analyzed 和 no。另外，对基于字符串的字段，也可以设置为 not_analyzed。如果设置为analyzed，该字段将被编入索引以供搜索。如果设置为 no，将无法搜索该字段。默认值为 analyzed。在基于字符串的字段中，还有一个额外的选项 not_analyzed。此设置意味着字段将不经分析而编入索引，**使用原始值被编入索引，在搜索的过程中必须全部匹配**。索引属性设置为 no 将使 include_in_all 属性失效。
- store：这个属性的值可以是 yes 或 no，指定了该字段的原始值是否被写入索引中。默认值设置为no，这意味着在结果中不能返回该字段（然而，如果你使用_source字段，即使没有存储也可返回返回这个值），但是如果该值编入索引，仍可以基于它来搜索数据。_
- boost：该属性的默认值是1。基本上，它定义了在文档中该字段的重要性。boost 的值越高，字段中值的重要性也越高。
- null_value：如果该字段并非索引文档的一部分，此属性指定应写入索引的值。默认的行为是忽略该字段。_
- copy_to：此属性指定一个字段，字段的所有值都将复制到该指定字段。_
- include_in_all：此属性指定该字段是否应包括在_all字段中。默认情况下，如果使用_all字段，所有字段都会包括

具体深入到各个类型特有属性值，请参考官方文档，这里就不一一指出了。

#### 使用分析器

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-analyzers.html)

对于字符串类型的字段，可以指定 Elasticsearch 应该使用哪个分析器。分析器是一个用于分析数据或以我们想要的方式查询数据的工具。例如，用空格和小写字符把单词隔开时，不必担心用户发送的单词是小写还是大写。Elasticsearch 使我们能够在索引和查询时使用不同的分析器，并且可以在搜索过程的每个阶段选择处理数据的方式。使用分析器时，只需在指定字段的正确属性上设置它的名字，就这么简单。

##### 开箱机用的分析器

Elasticsearch 允许我们使用众多默认定义的分析器中的一种。如下分析器可以开箱即用。

- [Standard Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-standard-analyzer.html)

  方便大多数欧洲语言的标准分析器。

- [Simple Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-simple-analyzer.html)

  基于非字母字符来分离所提供的值，并将其转换为小写形式。

- [Whitespace Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-whitespace-analyzer.html)

  基于空格

- [Stop Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-stop-analyzer.html)

  类似于 Simple，除了 Simple 提供的之外，还基于提供的停用词过滤数据

- [Keyword Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-keyword-analyzer.html)

  非常简单的分析器，只传入提供的值。你可以通过指定字段为 `not_analyzed` 来达到相同的目的。

- [Pattern Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-pattern-analyzer.html)

  通过正则来分离文本

- [Language Analyzers](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-lang-analyzer.html)

  特定语言环境下工作，支持多种语言。

- [Fingerprint Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-fingerprint-analyzer.html)

  基于自己创建的一个检测重复的 fingerprint 分析器。

##### 定义自己的分析器

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/analysis-custom-analyzer.html)

通过简单设置过滤器来定义自己的分析器

```json
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type":      "custom",
          "tokenizer": "standard",
          "char_filter": [
            "html_strip"
          ],
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  }
}
```

定义了一个 my_custom_analyzer 的分析器

### 扩展索引结构

#### 元字段

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/mapping-fields.html)

- `_id`

  存储着索引时设置的实际标识符

- `_uid`

  `_type` 与 `_id` 结合，文档唯一标识符

- `_type`

  文档类型

- `_all`

  存储其他字段中的数据以便于搜索。

- `_index`

  存储文档相关索引信息。

- `_source`

  可以生成索引过程中存储发送到 ElasticSearch 的原始 JSON 文档。

- `_size`

  自动索引 `_source` 字段的原始大小。

#### 嵌套结构

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/nested.html)

基本上，通过使用嵌套对象，Elasticsearch 允许我们连接一个主文档和多个附属文档。主文档及嵌套文档一同被索引，放置于索引的同一段上（实际在同一块上），确保为该数据结构获取最佳性能。更改文档也是一样的，除非使用更新API，你需要同时索引父文档和其他所有嵌套文档。

```json
PUT my_index/my_type/1
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```

```json
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.first": "Alice" }},
        { "match": { "user.last":  "Smith" }}
      ]
    }
  }
}
```

### 段合并

随着时间的推移和持续索引数据，越来越多的段被创建。因此，搜索性能可能会降低，而且索引可能比原先大，因为它仍含有被删除的文件。这使得段合并有了用武之地。

段合并的处理过程是：底层的 Lucene 库获取若干段，并在这些段信息的基础上创建一个新的段。由此产生的段拥有所有存储在原始段中的文档，除了被标记为删除的那些之外。合并操作之后，源段将从磁盘上删除。这是因为段合并在CPU和I/O的使用方面代价是相当高的，关键是要适当地控制这个过程被调用的时机和频率。

### 段合并的必要性

1. 构成索引的段越多，搜索速度越慢，需要使用的Lucene内存也越多。
2. 索引使用的磁盘空间和资源，例如文件描述符。如果从索引中删除许多文档，直到合并发生，则这些文档只是被标记为已删除，而没有在物理上删除。因而，大多数占用了CPU和内存的文档可能并不存在！

好在Elasticsearch使用合理的默认值做段合并，这些默认值很可能不再需要做任何更改。

#### 合并策略

三种策略：

- tiered：这是默认合并策略，合并尺寸大致相似的段，并考虑到每个层（tier）允许的最大段数量；
- log_byte_size：这个合并策略下，随着时间推移，将产生由索引大小的对数构成的索引，其中存在着一些较大的段以及一些合并因子较小的段等；
- log_doc：这个策略类似于log_byte_size合并策略，但根据索引中的文档数而非段的实际字节数来操作。

#### 合并调度器

指示 ElasticSearch 合并过程的方式，有如下可能：

- 并发合并调度器：这是默认的合并过程，在独立的线程中执行，定义好的线程数量可以并行合并。
- 串行合并调度器：这一合并过程在调用线程（即执行索引的线程）中执行。合并进程会一直阻塞线程直到合并完成。调度器可使用index.merge.scheduler.type参数设置。若要使用串行合并调度器，需把参数值设为serial；若要使用并发调度器，则需把参数值设为concurrent。

### 路由介绍

默认情况下，Elasticsearch 会在所有索引的分片中均匀地分配文档。然而，这并不总是理想情况。为了获得文档，Elasticsearch必须查询所有分片并合并结果。然而，如果你可以把数据按照一定的依据来划分（例如，客户端标识符），就可以使用一个强大的文档和查询分布控制机制：路由。简而言之，**它允许选择用于索引和搜索数据的分片。**

#### 默认索引过程

默认情况下，Elasticsearch 计算文档标识符的散列值，以此为基础将文档放置于一个可用的主分片上。接着，这些文档被重新分配至副本。

#### 默认搜索过程

一般而言，我们将查询发送到 Elasticsearch 的一个节点，Elasticsearch将会根据搜索类型来执行查询。这通常意味着它首先查询所有节点得到标识符和匹配文档的得分，接着发送一个内部查询，但仅发送到相关的分片（包含所需文档的分片），最后获取所需文档来构建响应。

如下图所示

![ES搜索过程](https://github.com/BingLau7/blog/blob/master/images/blog_40/ES-search-process.jpg?raw=true)

假使把单个用户的所有文档放置于单个分片之中，并对此分片查询，会出现什么情况？是否对性能来说不明智？不，这种操作是相当便利的，也正是路由所允许的。

#### 路由

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/search.html#search-routing)

要记住，使用路由时，你仍然应该为与路由值相同的值添加一个过滤器。这是因为，路由值的数量或许会比索引分片的数量多。因此，一些不同的属性值可以指向相同的分片，如果你忽略过滤，得到的数据并非是路由的单个值，而是特定分片中驻留的所有路由值。

##### 路由参数

最简单的方法（但并不总是最方便的一个）是使用路由参数来提供路由值。索引或查询时，你可以添加路由参数到HTTP，或使用你所选择的客户端库来设置。

添加到固定路由值中

```json
POST /twitter/tweet?routing=kimchy
{
    "user" : "kimchy",
    "postDate" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

从路由值中查询

```json
POST /twitter/tweet/_search?routing=kimchy
{
    "query": {
        "bool" : {
            "must" : {
                "query_string" : {
                    "query" : "some query string here"
                }
            },
            "filter" : {
                "term" : { "user" : "kimchy" }
            }
        }
    }
}
```

##### 路由字段

为每个发送到Elasticsearch的请求指定路由值并不方便。事实上，在索引过程中，Elasticsearch允许指定一个字段，用该字段的值作为路由值。这样只需要在查询时提供路由参数。为此，在类型定义中需要添加以下代码：

```json
"_routing": {
  "required": true,
  "path": "userId"
}
```

上述定义意味着需要提供路由值（`"required":true`属性），否则，索引请求将失败。除此之外，我们还指定了path属性，说明文档的哪个字段值应被设置为路由值，在上述示例中，我们使用了 userId 字段值。这两个参数意味着用于索引的每个文档都需要定义 userId 字段。

添加路由部分后，整个更新的映射文件将如下所示：

```json
{
  "mappings": {
  	"post": {
      "_routing": {
      	"required": true, 
      	"path": "userId"
      }, 
   	  "properties": {
      	"id": {"type": "long", "store": "yes", "precision_ step": "0"},
      	"name": {"type" :"string", "store": "yes", "index": "analyzed"}, 
      	"contents": {"type": "string", "store": "no", "index": "analyzed" },
      	"userId": {"type": "long", "store": "yes", "precision_ step": "0"} 
      } 
  	} 
  } 
}
```

如果想使用上述映射来建 post 索引可以这样:

```json
curl -XPOST 'localhost:9200/posts/post/ 1' -d '{
  "id": 1, 
  "name":" New post", 
  "contents": "New test post", 
  "userId": 1234567 
}'
```

这样 ElasticSearch 将使用 1234567 作为索引时的路由值。

## 其他查询

### 复合查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/compound-queries.html)

复合查询就是支持可以把多个查询连接起来，或者改变其他查询的行为。

#### 布尔查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-bool-query.html)

```json
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user" : "kimchy" }
      },
      "filter": {
        "term" : { "tag" : "tech" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tag" : "wow" } },
        { "term" : { "tag" : "elasticsearch" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
```

可以通过布尔查询来封装无限数量的查询，并通过下面描述的节点之一使用一个逻辑值来连接它们。

- should：被它封装的布尔查询可能被匹配，也可能不被匹配。被匹配的should节点数目由minimum_should_match参数控制。
- must：被它封装的布尔查询必须被匹配，文档才会返回。
- must_not：被它封装的布尔查询必须不被匹配，文档才会返回。

上述每个节点都可以在单个布尔查询中出现多次。这允许建立非常复杂的查询，有多个嵌套级别（在一个布尔查询中包含另一个布尔查询）。记住，结果文档的得分将由文档匹配的所有封装的查询得分总和计算得到。

除了上述部分以外，还可以在查询主体中添加以下参数控制其行为。

- boost：此参数指定了查询使用的加权值，默认为1.0。加权值越高，匹配文档的得分越高。
- minimum_should_match：此参数的值描述了文档被视为匹配时，应该匹配的should子句的最少数量。举例来说，它可以是个整数值，比如2，也可以是个百分比，比如75%。
- disable_coord：此参数的默认值为false，允许启用或禁用分数因子的计算，该计算是基于文档包含的所有查询词条。如果得分不必太精确，但要查询快点，那么应该将它设置为true。

#### 加权查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-boosting-query.html)

```json
GET /_search
{
    "query": {
        "boosting" : {
            "positive" : {
                "term" : {
                    "field1" : "value1"
                }
            },
            "negative" : {
                 "term" : {
                     "field2" : "value2"
                }
            },
            "negative_boost" : 0.2
        }
    }
}
```

加权查询封装了两个查询，并且降低其中一个查询返回文档的得分。加权查询中有三个节点需要定义：

- positive部分，包含所返回文档得分不会被改变的查询；
- negative部分，返回的文档得分将被降低；
- negative_boost部分，包含用来降低negative部分查询得分的加权值。

加权查询的优点是，positive 部分和 negative 部分包含的查询结果都会出现在搜索结果中，而某些查询的得分将被降低。如果使用布尔查询的 must_not 节点，将得不到这样的结果。

#### constant_score 查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-constant-score-query.html)

```json
GET /_search
{
    "query": {
        "constant_score" : {
            "filter" : {
                "term" : { "user" : "kimchy"}
            },
            "boost" : 1.2
        }
    }
}
```

constant_score 查询封装了另一个查询（或过滤），并为每一个所封装查询（或过滤）返回的文档返回一个常量得分。它**允许我们严格控制与一个查询或过滤匹配的文档得分**。

### function_score 查询

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-dsl-function-score-query.html)

```json
GET /_search
{
    "query": {
        "function_score": {
            "query": { "match_all": {} },
            "boost": "5",
            "random_score": {}, 
            "boost_mode":"multiply"
        }
    }
}
```

function_score 查询允许我们通过提供一些计算函数来修改检索文档的得分。

### 过滤器

#### 后过滤器

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/search-request-post-filter.html)

出于性能考虑，当你需要对搜索结果和聚合结果做不同的过滤时，你才应该使用 `post_filter`， `post_filter` 的特性是在查询之后 执行，任何过滤对性能带来的好处（比如缓存）都会完全失去。

在我们需要不同过滤时， `post_filter` 只与聚合一起使用。

在任何搜索中使用过滤器，只需在于 query 节点相同级别上添加一个 filter 节点。如果你只想要过滤器，也可以完全忽略 query 节点。示例，搜索 title 字段并向其添加过滤器：

```json
{
  "query": {
      "match": {"title": "Catch-22"}
  },
  "post_filter": {
      "term": {"year": 1961}
  }
}
```

其返回结果会缩小到过滤器指定的范围中。

#### 过滤器

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/query-filter-context.html)

绝大部分字段与 query 相同，具体可参照文档。

#### filter or post_filter

[资料](https://stackoverflow.com/questions/32085557/elasticsearch-post-filter-or-filter)

> To sum it up
>
> - a `filtered` query affects **both search results and aggregations**
> - while a `post_filter` **only affects the search results but NOT the aggregations**

### 排序

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/search-request-sort.html)

默认是以 `_score` 的倒叙排序的

#### 指定字段排序

```json
{
    "sort" : [
        { "post_date" : {"order" : "asc"}},
        "user",
        { "name" : "desc" },
        { "age" : "desc" },
        "_score"
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

类似于指定在 `sore` 里面的 array 字段。排序数组中的字段，如果字段相同就采用下一个字段来确定顺序。

> The order defaults to `desc` when sorting on the `_score`, and defaults to `asc` when sorting on anything else.

#### 指定缺少字段行为

如果排序字段有缺失可以指定为第一个(`_first`)或最后一个(`_last`)

```json
{
    "sort" : [
        { "price" : {"missing" : "_last"} }
    ],
    "query" : {
        "term" : { "product" : "chocolate" }
    }
}
```

#### 动态排序

ElasticSearch 允许我们使用具有多个值得字段进行排序，可以使用脚本来控制排序比较告诉 ElasticSearch 如果计算应用于排序的值来达到目的。

```json
{
    "query" : {
        "term" : { "user" : "kimchy" }
    },
    "sort" : {
        "_script" : {
            "type" : "number",
            "script" : {
                "lang": "painless",
                "source": "doc['field_name'].value * params.factor",
                "params" : {
                    "factor" : 1.1
                }
            },
            "order" : "asc"
        }
    }
}
```

### Explain

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-explain.html)

ElasticSearch 的 Explain API 会给出查询的文档的具体计算解释，无论这个文档是否匹配到了这条查询。

```json
GET /twitter/tweet/0/_explain
{
      "query" : {
        "match" : { "message" : "elasticsearch" }
      }
}
```

```json
{
   "_index": "twitter",
   "_type": "tweet",
   "_id": "0",
   "matched": true,
   "explanation": {
      "value": 1.6943599,
      "description": "weight(message:elasticsearch in 0) [PerFieldSimilarity], result of:",
      "details": [
         {
            "value": 1.6943599,
            "description": "score(doc=0,freq=1.0 = termFreq=1.0\n), product of:",
            "details": [
               {
                  "value": 1.3862944,
                  "description": "idf, computed as log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5)) from:",
                  "details": [
                     {
                        "value": 1.0,
                        "description": "docFreq",
                        "details": []
                     },
                     {
                        "value": 5.0,
                        "description": "docCount",
                        "details": []
                      }
                   ]
               },
                {
                  "value": 1.2222223,
                  "description": "tfNorm, computed as (freq * (k1 + 1)) / (freq + k1 * (1 - b + b * fieldLength / avgFieldLength)) from:",
                  "details": [
                     {
                        "value": 1.0,
                        "description": "termFreq=1.0",
                        "details": []
                     },
                     {
                        "value": 1.2,
                        "description": "parameter k1",
                        "details": []
                     },
                     {
                        "value": 0.75,
                        "description": "parameter b",
                        "details": []
                     },
                     {
                        "value": 5.4,
                        "description": "avgFieldLength",
                        "details": []
                     },
                     {
                        "value": 3.0,
                        "description": "fieldLength",
                        "details": []
                     }
                  ]
               }
            ]
         }
      ]
   }
}
```

看起来有点复杂，这里最重要的内容就是对文档计算得到的总分，如果总分等于0，则该文档将不能匹配给定的查询。另一个重要内容是关于不同打分项的描述信息。根据查询类型的不同，打分项会以不同方式对最后得分产生影响。

## 搜索进阶

### Apache Lucene 评分简介

[官方文档](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/scoring-theory.html#tfidf)

TF/IDF 算法的实用计算公式如下：
$$
score(q, d) = coord(q, d) * queryNorm(q) * \sum_{tinq} (tf(tind)*idf(t)^2 * boot(t) * norm(t,d))
$$
其中 q 是查询 d 是文档。还有两个不直接依赖于查询词条的因子 coord 和 queryNorm。

公式中这两个元素跟查询中的每个词计算而得的总和相乘。另一方面，该总和由给定词的词频、逆文档频率、词条加权和规范（长度规范）相乘而来。

上述规则的好处是，你不需要记住全部内容。应该知道的是影响文档评分的因素。下面是一些派生自上述等式的规则

#### 派生规则

- 匹配的词条越罕见，文档的得分越高；
- 文档的字段越小，文档的得分越高；
- 字段的加权越高，文档的得分越高；
- 我们可以看到，文档匹配的查询词条数目越高、字段越少（意味着索引的词条越少），Lucene给文档的分数越高。同时，罕见词条比常见词条更受评分的青睐。

### ElasticSearch 脚本功能

Elasticsearch有几个可以使用脚本的功能。你已经看过一些例子，如更新文件、过滤和搜索。
Elasticsearch使用脚本执行的任何请求中，我们会注意到以下相似的属性。

- script: 实际包含的脚本代码
- lang: 定义实用的脚本语言，默认为 mvel
- params: 此对象包含参数及其值。每个定义的参数可以通过指定参数名称在脚本中使用。通过使用参数，我们可以编写更干净的代码。由于可以缓存，使用参数的脚本比嵌入常数的代码执行得更快。

实例：排序的脚本功能

```json
{
  "function_score": {
    "functions": [
      { ...location clause... }, 
      { ...price clause... }, 
      {
        "script_score": {
          "params": { 
            "threshold": 80,
            "discount": 0.1,
            "target": 10
          },
          "script": "price  = doc['price'].value; margin = doc['margin'].value;
          if (price < threshold) { return price * margin / target };
          return price * (1 - discount) * margin / target;" 
        }
      }
    ]
  }
}
```

#### 脚本执行中可用对象

在不同的操作过程中，Elasticsearch允许在脚本中使用不同的对象。比如，在搜索过程中，下列对象是可用的。

- _doc（也可以用doc）：这是个 `org.elasticsearch.search.lookup.DocLookup` 对象的实例。通过它可以访问当前找到的文档，附带计算的得分和字段的值。_
- source：这是个 `org.elasticsearch.search.lookup.SourceLookup`对象的实例，通过它可以访问当前文档的 source，以及定义在 source 中的值。
- fields：这是个 `org.elasticsearch.search.lookup.FieldsLookup` 对象的实例，通过它可以访问文档的所有字段。

另一方面，在文档更新过程中，Elasticsearch 只通过 _source 属性公开了 ctx 对象，通过它可以访问当前文档。

我们之前看到过，在文档字段和字段值的上下文中提到了几种方法。现在让我们通过下面的例子，看看如何获取title字段的值。

在括号中，你可以看到 Elasticsearch 从 library 索引中为我们的一个示例文档返回的值：

- doc.title.value（crime）；
- source.title（CrimeandPunishment）；
- fields.title.value（null）。

有点疑惑，不是吗？在索引期间，一个字段值作为 source 文档的一部分被发送到 Elasticsearch。Elasticsearch 可以存储此信息，而且**默认的就是存储**。此外，文档被解析，**每个被标记成 stored 的字段可能都存储在索引中**（也就是说，store 属性设置为 true；否则，默认情况下，字段不存储）。最后，字段值可以配置成 indexed。这意味着分析该字段值，划分为标记，并放置在索引中。

综上所述，一个字段可能以如下方式存储在索引中：

- 作为_source文档的一部分；
- 一个存储并未经解析的值；
- 一个解析成若干标记的值。

#### 使用其他脚本语言

[官方文档](https://www.elastic.co/guide/en/elasticsearch/plugins/5.6/scripting.html)

### 聚合

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/search-aggregations.html)

提供基于搜索查询的聚合功能，它基于简单的结构建立而成，可用进行组合以便于构建复杂的数据，所以称为聚合。聚合可以被看作是在一组文档上（查询得到）分析得到的信息的结果。 

#### [*Metric*](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/search-aggregations-metrics.html) 度量聚合

##### min、max、sum、avg 聚合

min、max、sum 和 avg 聚合的使用很相似。它们对于给定字段分别返回最小值、最大值、总和和平均值。任何数值型字段都可以作为这些值的源。比如下面通过 `grade` 计算平均值然后返回字段标识为 `avg_grade`

```json
POST /exams/_search?size=0
{
    "aggs" : {
        "avg_grade" : { "avg" : { "field" : "grade" } }
    }
}

# response
{
    ...
    "aggregations": {
        "avg_grade": {
            "value": 75.0
        }
    }
}
```

##### value count

value_count 聚合跟前面描述的聚合类似，只是输入字段不一定要是数值型的。

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "types_count" : { "value_count" : { "field" : "type" } }
    }
}

# response
{
    ...
    "aggregations": {
        "types_count": {
            "value": 7
        }
    }
}
```

##### 脚本聚合

使用 script 做到 value_count 聚合

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "type_count" : {
            "value_count" : {
                "script" : {
                    "source" : "doc['type'].value"
                }
            }
        }
    }
}
```

##### stats 和 extended_status(更多信息) 聚合

stats 和 extended_stats 聚合可以看成是在单一聚合对象中返回所有前面描述聚合的一种聚合。

```json
POST /exams/_search?size=0
{
    "aggs" : {
        "grades_stats" : { "stats" : { "field" : "grade" } }
    }
}

# response
{
    ...

    "aggregations": {
        "grades_stats": {
            "count": 2,
            "min": 50.0,
            "max": 100.0,
            "avg": 75.0,
            "sum": 150.0
        }
    }
}
```

```json
GET /exams/_search
{
    "size": 0,
    "aggs" : {
        "grades_stats" : { "extended_stats" : { "field" : "grade" } }
    }
}

# response
{
    ...

    "aggregations": {
        "grades_stats": {
           "count": 2,
           "min": 50.0,
           "max": 100.0,
           "avg": 75.0,
           "sum": 150.0,
           "sum_of_squares": 12500.0,
           "variance": 625.0,
           "std_deviation": 25.0,
           "std_deviation_bounds": {
            "upper": 125.0,
            "lower": 25.0
           }
        }
    }
}
```

更多查看文档

#### [*Bucketing*](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/search-aggregations-bucket.html) 桶聚合

桶聚合返回很多子集，并限定输入数据到一个特殊的叫做桶的子集中。

##### terms 聚合

terms 聚合为字段中每个词条返回一个桶。这允许你对生成字段每个值得统计。

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : { "field" : "genre" }
        }
    }
}

# response
{
    ...
    "aggregations" : {
        "genres" : {
            "doc_count_error_upper_bound": 0, 
            "sum_other_doc_count": 0, 
            "buckets" : [ 
                {
                    "key" : "electronic",
                    "doc_count" : 6
                },
                {
                    "key" : "rock",
                    "doc_count" : 3
                },
                {
                    "key" : "jazz",
                    "doc_count" : 2
                }
            ]
        }
    }
}
```

##### range 聚合

range 聚合使用定义的范围来创建桶。

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "ranges" : [
                    { "to" : 100.0 },
                    { "from" : 100.0, "to" : 200.0 },
                    { "from" : 200.0 }
                ]
            }
        }
    }
}

# response
{
    ...
    "aggregations": {
        "price_ranges" : {
            "buckets": [
                {
                    "key": "*-100.0",
                    "to": 100.0,
                    "doc_count": 2
                },
                {
                    "key": "100.0-200.0",
                    "from": 100.0,
                    "to": 200.0,
                    "doc_count": 2
                },
                {
                    "key": "200.0-*",
                    "from": 200.0,
                    "doc_count": 3
                }
            ]
        }
    }
}
```

##### date_range 聚合

date_range 聚合类似于 range，但它专用在使用日期类型的字段。

```json
POST /sales/_search?size=0
{
    "aggs": {
        "range": {
            "date_range": {
                "field": "date",
                "format": "MM-yyy",
                "ranges": [
                    { "to": "now-10M/M" }, 
                    { "from": "now-10M/M" } 
                ]
            }
        }
    }
}

# response
{
    ...
    "aggregations": {
        "range": {
            "buckets": [
                {
                    "to": 1.4436576E12,
                    "to_as_string": "10-2015",
                    "doc_count": 7,
                    "key": "*-10-2015"
                },
                {
                    "from": 1.4436576E12,
                    "from_as_string": "10-2015",
                    "doc_count": 0,
                    "key": "10-2015-*"
                }
            ]
        }
    }
}
```

##### missing 聚合

基于字段数据的单个桶集合，创建当前文档集上下文中缺少字段值（实际上缺少字段或设置了配置的 NULL 值）的所有文档的桶。 此聚合器通常会与其他字段数据存储桶聚合器（如 range）一起使用，以返回由于缺少字段数据值而无法放置在其他存储桶中的所有文档的信息。

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "products_without_a_price" : {
            "missing" : { "field" : "price" }
        }
    }
}

# response
{
    ...
    "aggregations" : {
        "products_without_a_price" : {
            "doc_count" : 00
        }
    }
}
```

##### nested 聚合

针对 nested 结构的聚合

```json
GET /_search
{
    "query" : {
        "match" : { "name" : "led tv" }
    },
    "aggs" : {
        "resellers" : {
            "nested" : {
                "path" : "resellers" # nested
            },
            "aggs" : {
                "min_price" : { "min" : { "field" : "resellers.price" } }
            }
        }
    }
}

# response
{
  ...
  "aggregations": {
    "resellers": {
      "doc_count": 0,
      "min_price": {
        "value": 350
      }
    }
  }
}
```

##### histogram 聚合

想象为柱状图一样，针对某个 field 的多个值进行聚合。

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "prices" : {
            "histogram" : {
                "field" : "price",
                "interval" : 50
            }
        }
    }
}

# response
{
    ...
    "aggregations": {
        "prices" : {
            "buckets": [
                {
                    "key": 0.0,
                    "doc_count": 1
                },
                {
                    "key": 50.0,
                    "doc_count": 1
                },
                {
                    "key": 100.0,
                    "doc_count": 0
                },
                {
                    "key": 150.0,
                    "doc_count": 2
                },
                {
                    "key": 200.0,
                    "doc_count": 3
                }
            ]
        }
    }
}
```

跟多其他的聚合可以查看官方文档.

### 建议器

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/search-suggesters.html)

我们可以把建议器看成这样的功能：在考虑性能的情况下，允许纠正用户的拼写错误，以及构建一个自动完成功能。

#### 建议器类型

- term：更正每个传入的单词，在非短语查询中很有用。
- phrase：工作在短语上，返回一个恰当的短语。
- completion：在索引中存储复杂的数据结构，提供快速高效的自动完成功能。
- context：配合 completion 使用，给其搜索文档字段提供某些上下文信息以供更好的进行 completion

#### 使用

如果我们需要在查询结果中得到建议，例如使用 `match` 查询并尝试为 tring out Elasticsearc 短语得到一个建议，该短语包含一个拼写错误的词条。为此我们可以：

```json
POST twitter/_search
{
  "query" : {
    "match": {
      "message": "tring out Elasticsearch"
    }
  },
  "suggest" : {
    "my-suggestion" : {
      "text" : "trying out Elasticsearch",
      "term" : {
        "field" : "message"
      }
    }
  }
}
```

如果希望为同样的文本得到多个建议，则可把建议嵌入 suggest 对象中，并把 text 属性设置为选择的建议对象。（省略 query 结构）

```json
POST _search
{
  "suggest": {
    "my-suggest-1" : {
      "text" : "tring out Elasticsearch",
      "term" : {
        "field" : "message"
      }
    },
    "my-suggest-2" : {
      "text" : "kmichy",
      "term" : {
        "field" : "user"
      }
    }
  }
}

# response
{
  "took" : 105,
  "timed_out" : false,
  "_shards" : {
	...
  },
  "hits" : {
    []
  },
  "suggest" : {
    "my-suggest-1" : [
      {
        "text" : "tring",  // 原始单测
        "offset" : 0,      // 原本的偏移值
        "length" : 5,      // 单词长度
        "options" : [      // 建议
          {
            "text" : "trying",  // 建议的文本
            "score" : 0.8,      // 建议的得分，越高越好
            "freq" : 2          // 建议的频率，代表我们执行建议查询的索引上，该单词出现在文档中的次数
          }
        ]
      },
      {
        "text" : "out",
        "offset" : 6,
        "length" : 3,
        "options" : [ ]
      },
      {
        "text" : "elasticsearch",
        "offset" : 10,
        "length" : 13,
        "options" : [ ]
      }
    ],
    "my-suggest-2" : [
      {
        "text" : "kmichy",
        "offset" : 0,
        "length" : 6,
        "options" : [
          {
            "text" : "kimchy",
            "score" : 0.8333333,
            "freq" : 2
          }
        ]
      }
    ]
  }
}
```

这里 suggest 的命名只是示例，最好使用有意义的命名。

#### 建议器公用配置

- text：这个选项定义了我们希望得到建议的文本。此参数是必须的。
- field：这是另一个必须提供的参数。field参数设置了为哪个字段生成建议。
- analyzer：这个选项定义了分析器的名字，该分析器用作分析text参数提供的文本。如果未设置，Elasticsearch将使用field参数所指定的字段所用的分析器。
- size：这个参数默认为5，指定了text参数中每个词条可以返回的建议的最大数字。
- sort：此选项允许指定 Elasticsearch 返回的建议如何排序。默认情况下，此选项设置成 score，Elasticsearch将首先按照建议的得分排，然后按文档频率，最后按词条排。第二个可能值为 frequency，意味着结果首先按文档频率排，然后按分数，最后按词条。

具体到各个建议器的使用推荐查看文档，玩法太多了。
