**简介：** 数据模型是抽象描述现实世界的一种工具和方法，是通过抽象的实体及实体之间联系的形式，用图形化的形式去描述业务规则的过程，从而表示现实世界中事务的相互关系的一种映射。

## 1、什么是数据模型？

数据模型是抽象描述现实世界的一种工具和方法，是通过抽象的实体及实体之间联系的形式，用图形化的形式去描述业务规则的过程，从而表示现实世界中事务的相互关系的一种映射。

**核心概念**：

1.  实体：现实世界中存在的可以相互区分的事务或概念称为实体。  
    实体可以分为事物实体和概念实体。例如：一个学生、一个程序员等是事物实体。一门课、一个班级等称为概念实体。
2.  实体的属性：每个实体都有自己的特征，利用实体的属性可以区别不同的实体。例如。学生实体的属性为姓名、性别、年龄等。

## 2、数据建模的过程？

数据建模大致分为三个阶段，概念建模阶段，逻辑建模阶段和物理建模阶段。

![image.png](https://ucc.alicdn.com/pic/developer-ecology/f05bd1f2c1dd4f6aaf5054b422ec74e6.png "image.png")

### **2.1 概念建模阶段**

概念建模阶段，主要做三件事：

-   客户交流
-   理解需求
-   形成实体

确定系统的核心需求和范围边界，设计实体与实体之间的关系。  
在概念建模阶段，我们只需要关注实体即可，不用关注任何实现细节。很多人都希望在这个阶段把具体表结构，索引，约束，甚至是存储过程都想好，没必要！因为这些东西使我们在物理建模阶段需要考虑的东西，这个时候考虑还为时尚早。  
概念模型在整个数据建模时间占比：10%左右。

### **2.2 逻辑建模阶段**

逻辑建模阶段，主要做二件事：

-   进一步树立业务需求，
-   确定每个实体的属性、关系和约束等。  
    逻辑模型是对概念模型的进一步分解和细化，描述了实体、实体属性以及实体之间的关系，是概念模型延伸，一般的逻辑模型有第三范式，星型模型和雪花模型。模型的主要元素为主题、实体、实体属性和关系。

逻辑模型的作用主要有两点。

-   一是便于技术开发人员和业务人员或者用户进行沟通 交流，使得整个概念模型更易于理解，进一步明确需求。
-   二是作为物理模型设计的基础，由于逻辑模型不依赖于具体的数据库实现，使用逻辑模型可以生成针对具体 数据库管理系统的物理模型，保证物理模型充分满足用户的需求。  
    逻辑模型在整个数据建模时间占比：60—70%左右。

### **2.3 物理建模阶段**

物理建模阶段，主要做一件事：

-   结合具体的数据库产品（mysql/oracle/mongo/elasticsearch），在满足业务读写性能等需求的前提下确定最终的定义。  
    物理模型是在逻辑模型的基础上描述模型实体的细节，包括数据库产品对应的数据类型、长度、索引等因素，为逻辑模型选择一个最有的物理存储环境。
-   逻辑模型转化为物理模型的过程也就是实体名转化为表名，属性名转化为物理列名的过程。  
    在设计物理模型时，还需要考虑数据存储空间的分配，包括对列属性必须做出明确的定 义。

> 例如：客户姓名的数据类型是varchar2，长度是20，存储在Oracle数据库中，并且建立索引用于提高该字段的查询效率。

物理模型在整个数据建模时间占比：20—30%左右。

## 3、数据建模的意义？

![image.png](https://ucc.alicdn.com/pic/developer-ecology/8c36b9905f4c45a1b2a173d637b4129a.png "image.png")

**如下图所示**：

![image.png](https://ucc.alicdn.com/pic/developer-ecology/21711278080d47abaedab29258522b96.png "image.png")

数据模型支撑了系统和数据，系统和数据支撑了业务系统。

一个好的数据模型：

-   能让系统更好的集成、能简化接口。
-   能简化数据冗余、减少磁盘空间、提升传输效率。
-   兼容更多的数据，不会因为数据类型的新增而导致实现逻辑更改。
-   能帮助更多的业务机会，提高业务效率。
-   能减少业务风险、降低业务成本。

> 举例: 借助logstash实现mysql到Elasticsearch的增量同步，如果数据建模阶段没有设计：时间戳或者自增ID，就几乎无法实现。

## 4、Elasticsearch数据建模注意事项

![image.png](https://ucc.alicdn.com/pic/developer-ecology/1d13909dce064fd996378a79b28ddc34.png "image.png")

### **4.1 ES Mapping 设置**

![image.png](https://ucc.alicdn.com/pic/developer-ecology/3731d014bbce46749d9e38f671ffc31e.png "image.png")

### **4.2 ES Mapping 字段设置流程图**

![image.png](https://ucc.alicdn.com/pic/developer-ecology/5e7d766f9a684ab2ba3d8900f547b24e.png "image.png")

### **4.3 ES 万能Mapping 模板参考**

以下的索引 Mapping中，\_source设置为false，同时各个字段的store根据需求设置了true和false。  
url的doc_values设置为false，该字段url不用于聚合和排序操作。

```
PUT blog_index
{
  "mappings": {
    "doc": {
      "_source": {
        "enabled": false
      },
      "properties": {
        "title": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 100
            }
          },
          "store": true
        },
        "publish_date": {
          "type": "date",
          "store": true
        },
        "author": {
          "type": "keyword",
          "ignore_above": 100, 
          "store": true
        },
        "abstract": {
          "type": "text",
          "store": true
        },
        "content": {
          "type": "text",
          "store": true
        },
        "url": {
          "type": "keyword",
          "doc_values":false,
          "norms":false,
          "ignore_above": 100, 
          "store": true
        }
      }
    }
  }
}
```

## 5、不可回避——ES多表关联

> 实际业务问题：多层数据结构，一对多关系，如何用一个查询查询所有的数据？  
> 比如数据结构如下：帖子–帖子评论–评论用户 3层。  
> 现在需要查询一条帖子，最好能查询到帖子下的评论，还有评论下面的用户数据，一个查询能搞定吗？目前两层我可以查询到，3层就不行了。  
> 如果一次查询不到，那如何设计数据结构？又应该如何查询呢？

目前ES主要有以下4种常用的方法来处理数据实体间的关联关系：

### **（1）Application-side joins（服务端Join或客户端Join）**

这种方式，索引之间完全独立（利于对数据进行标准化处理，如便于上述两种增量同步的实现），由应用端的多次查询来实现近似关联关系查询。这种方法适用于第一个实体只有少量的文档记录的情况（使用ES的terms查询具有上限，默认1024，具体可在elasticsearch.yml中修改），并且最好它们很少改变。这将允许应用程序对结果进行缓存，并避免经常运行第一次查询。

### **（2）Data denormalization（数据的非规范化）**

这种方式，通俗点就是通过字段冗余，以一张大宽表来实现粗粒度的index，这样可以充分发挥扁平化的优势。但是这是以牺牲索引性能及灵活度为代价的。使用的前提：冗余的字段应该是很少改变的；比较适合与一对少量关系的处理。当业务数据库并非采用非规范化设计时，这时要将数据同步到作为二级索引库的ES中，就很难使用上述增量同步方案，必须进行定制化开发，基于特定业务进行应用开发来处理join关联和实体拼接。

ps：宽表处理在处理一对多、多对多关系时，会有字段冗余问题，适合“一对少量”且这个“一”更新不频繁的应用场景。宽表化处理，在查询阶段如果只需要“一”这部分时，需要进行结果去重处理（可以使用ES5.x的字段折叠特性，但无法准确获取分页总数，产品设计上需采用上拉加载分页方式）

### **（3）Nested objects（嵌套文档）**

索引性能和查询性能二者不可兼得，必须进行取舍。嵌套文档将实体关系嵌套组合在单文档内部（类似与json的一对多层级结构），这种方式牺牲索引性能（文档内任一属性变化都需要重新索引该文档）来换取查询性能，可以同时返回关系实体，比较适合于一对少量的关系处理。  
ps: 当使用嵌套文档时，使用通用的查询方式是无法访问到的，必须使用合适的查询方式（nested query、nested filter、nested facet等），很多场景下，使用嵌套文档的复杂度在于索引阶段对关联关系的组织拼装。

### **（4）Parent/child relationships（父子文档）**

父子文档牺牲了一定的查询性能来换取索引性能，适用于一对多的关系处理。其通过两种type的文档来表示父子实体，父子文档的索引是独立的。父-子文档ID映射存储在 Doc Values 中。当映射完全在内存中时， Doc Values 提供对映射的快速处理能力，另一方面当映射非常大时，可以通过溢出到磁盘提供足够的扩展能力。 在查询parent-child替代方案时，发现了一种filter-terms的语法，要求某一字段里有关联实体的ID列表。基本的原理是在terms的时候，对于多项取值，如果在另外的index或者type里已知主键id的情况下，某一字段有这些值，可以直接嵌套查询。具体可参考官方文档的示例：通过用户里的粉丝关系，微博和用户的关系，来查询某个用户的粉丝发表的微博列表。  
ps：父子文档相比嵌套文档较灵活，但只适用于“一对大量”且这个“一”不是海量的应用场景，该方式比较耗内存和CPU，这种方式查询比嵌套方式慢5~10倍，且需要使用特定的has_parent和has_child过滤器查询语法，查询结果不能同时返回父子文档（一次join查询只能返回一种类型的文档）。而受限于父子文档必须在同一分片上，ES父子文档在滚动索引、多索引场景下对父子关系存储和联合查询支持得不好，而且子文档type删除比较麻烦（子文档删除必须提供父文档ID）。

如果业务端对查询性能要求很高的话，还是建议使用宽表化处理\*的方式，这样也可以比较好地应对聚合的需求。在索引阶段需要做join处理，查询阶段可能需要做去重处理，分页方式可能也得权衡考虑下。

## 6、小结

本篇文章基于rockybean《Elasticsearch从入门到实践》数据建模篇结合社区精彩问答进行了梳理和扩展，“站在巨人的”肩上，更能体会建模的重要性。  
实际业务开发中，务必重视建模，前期在建模方面多下苦功夫、后期的业务系统开发才能水到渠成，更健壮、更有扩展性！



# 字段折叠

一个普遍的需求是需要通过特定字段进行分组。例如我们需要按照用户名称 _分组_ 返回最相关的博客文章。 按照用户名分组意味着进行 `terms` 聚合。 为能够按照用户 _整体_ 名称进行分组，名称字段应保持 `not_analyzed` 的形式， 具体说明参考 [聚合与分析](https://www.elastic.co/guide/cn/elasticsearch/guide/current/aggregations-and-analysis.html "聚合与分析")：

```json
PUT /my_index/_mapping/blogpost
{
  "properties": {
    "user": {
      "properties": {
        "name": { 
          "type": "string",
          "fields": {
            "raw": { 
              "type":  "string",
              "index": "not_analyzed"
            }
          }
        }
      }
    }
  }
}
```

`user.name` 字段将用来进行全文检索。
`user.name.raw` 字段将用来通过 `terms` 聚合进行分组。

然后添加一些数据:

```json
PUT /my_index/user/1
{
  "name": "John Smith",
  "email": "john@smith.com",
  "dob": "1970/10/24"
}

PUT /my_index/blogpost/2
{
  "title": "Relationships",
  "body": "It's complicated...",
  "user": {
    "id": 1,
    "name": "John Smith"
  }
}

PUT /my_index/user/3
{
  "name": "Alice John",
  "email": "alice@john.com",
  "dob": "1979/01/04"
}

PUT /my_index/blogpost/4
{
  "title": "Relationships are cool",
  "body": "It's not complicated at all...",
  "user": {
    "id": 3,
    "name": "Alice John"
  }
}
```

现在我们来查询标题包含 `relationships` 并且作者名包含 `John` 的博客，查询结果再按作者名分组，感谢 [`top_hits` aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-aggregations-metrics-top-hits-aggregation.html) 提供了按照用户进行分组的功能：

```json
GET /my_index/blogpost/_search
{
  "size" : 0, 
  "query": { 
    "bool": {
      "must": [
        { "match": { "title":     "relationships" }},
        { "match": { "user.name": "John"          }}
      ]
    }
  },
  "aggs": {
    "users": {
      "terms": {
        "field":   "user.name.raw",      
        "order": { "top_score": "desc" } 
      },
      "aggs": {
        "top_score": { "max":      { "script":  "_score"           }}, 
        "blogposts": { "top_hits": { "_source": "title", "size": 5 }}
      }
    }
  }
}
```

我们感兴趣的博客文章是通过 `blogposts` 聚合返回的，所以我们可以通过将 `size` 设置成 0 来禁止 `hits` 常规搜索。
`query` 返回通过 `relationships` 查找名称为 `John` 的用户的博客文章。
`terms` 聚合为每一个 `user.name.raw` 创建一个桶。
`top_score` 聚合对通过 `users` 聚合得到的每一个桶按照文档评分对词项进行排序。
`top_hits` 聚合仅为每个用户返回五个最相关的博客文章的 `title` 字段。

这里显示简短响应结果：

```json
...
"hits": {
  "total":     2,
  "max_score": 0,
  "hits":      [] 
},
"aggregations": {
  "users": {
     "buckets": [
        {
           "key":       "John Smith", 
           "doc_count": 1,
           "blogposts": {
              "hits": { 
                 "total":     1,
                 "max_score": 0.35258877,
                 "hits": [
                    {
                       "_index": "my_index",
                       "_type":  "blogpost",
                       "_id":    "2",
                       "_score": 0.35258877,
                       "_source": {
                          "title": "Relationships"
                       }
                    }
                 ]
              }
           },
           "top_score": { 
              "value": 0.3525887727737427
           }
        },
...
```
因为我们设置 `size` 为 0 ，所以 `hits` 数组是空的。
在顶层查询结果中出现的每一个用户都会有一个对应的桶。
在每个用户桶下面都会有一个 `blogposts.hits` 数组包含针对这个用户的顶层查询结果。
用户桶按照每个用户最相关的博客文章进行排序。

使用 `top_hits` 聚合等效执行一个查询返回这些用户的名字和他们最相关的博客文章，然后为每一个用户执行相同的查询，以获得最好的博客。但前者的效率要好很多。

每一个桶返回的顶层查询命中结果是基于最初主查询进行的一个轻量 _迷你查询_ 结果集。这个迷你查询提供了一些你期望的常用特性，例如高亮显示以及分页功能。

# 文档并发问题
问题的原因是 Elasticsearch 不支持 [ACID 事务](http://en.wikipedia.org/wiki/ACID_transactions)。 对单个文件的变更是 ACIDic 的，但包含多个文档的变更不支持。

如果你的主要数据存储是关系数据库，并且 Elasticsearch 仅仅作为一个搜索引擎 或一种提升性能的方法，可以首先在数据库中执行变更动作，然后在完成后将这些变更复制到 Elasticsearch。 通过这种方式，你将受益于数据库 ACID 事务支持，并且在 Elasticsearch 中以正确的顺序产生变更。 并发在关系数据库中得到了处理。

如果你不使用关系型存储，这些并发问题就需要在 Elasticsearch 的事务水准进行处理。 以下是三个切实可行的使用 Elasticsearch 的解决方案，它们都涉及某种形式的锁：

-   全局锁
-   文档锁
-   树锁

但是目前系统开发方面我不想考虑es的锁，因为  
这三个方案—​全局、文档或树锁—​都没有处理锁最棘手的问题：如果持有锁的进程死了怎么办？

一个进程的意外死亡给我们留下了2个问题：

-   我们如何知道我们可以释放的死亡进程中所持有的锁？
-   我们如何清理死去的进程没有完成的变更？

所以目前对于我这边同步的做法是，在binlog同步中通过对 if xxx字段=old值 {xxx字段=新值},并且这个操作通过单个协程来操作,如果非要加锁，通过redis或者etcd锁

# 嵌套对象

由于在 Elasticsearch 中单个文档的增删改都是原子性操作,那么将相关实体数据都存储在同一文档中也就理所当然。 比如说,我们可以将订单及其明细数据存储在一个文档中。又比如,我们可以将一篇博客文章的评论以一个 `comments` 数组的形式和博客文章放在一起：

```json
PUT /my_index/blogpost/1
{
  "title": "Nest eggs",
  "body":  "Making your money work...",
  "tags":  [ "cash", "shares" ],
  "comments": [
    {
      "name":    "John Smith",
      "comment": "Great article",
      "age":     28,
      "stars":   4,
      "date":    "2014-09-01"
    },
    {
      "name":    "Alice White",
      "comment": "More like this please",
      "age":     31,
      "stars":   5,
      "date":    "2014-10-22"
    }
  ]
}
```
如果我们依赖[字段自动映射](https://www.elastic.co/guide/cn/elasticsearch/guide/current/dynamic-mapping.html "动态映射"),那么 `comments` 字段会自动映射为 `object` 类型。

由于所有的信息都在一个文档中,当我们查询时就没有必要去联合文章和评论文档,查询效率就很高。

但是当我们使用如下查询时,上面的文档也会被当做是符合条件的结果：
```json
GET /_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "Alice" }},
        { "match": { "age":  28      }} 
      ]
    }
  }
}
```
Alice实际是31岁,不是28!

正如我们在 [对象数组](https://www.elastic.co/guide/cn/elasticsearch/guide/current/complex-core-fields.html#object-arrays "内部对象数组") 中讨论的一样,出现上面这种问题的原因是 JSON 格式的文档被处理成如下的扁平式键值对的结构。

```json
{
  "title":            [ eggs, nest ],
  "body":             [ making, money, work, your ],
  "tags":             [ cash, shares ],
  "comments.name":    [ alice, john, smith, white ],
  "comments.comment": [ article, great, like, more, please, this ],
  "comments.age":     [ 28, 31 ],
  "comments.stars":   [ 4, 5 ],
  "comments.date":    [ 2014-09-01, 2014-10-22 ]
}
```

`Alice` 和 31 、 `John` 和 `2014-09-01` 之间的相关性信息不再存在。虽然 `object` 类型 (参见 [内部对象](https://www.elastic.co/guide/cn/elasticsearch/guide/current/complex-core-fields.html#inner-objects "多层级对象")) 在存储 _单一对象_ 时非常有用,但对于对象数组的搜索而言,毫无用处。

_嵌套对象_ 就是来解决这个问题的。将 `comments` 字段类型设置为 `nested` 而不是 `object` 后,每一个嵌套对象都会被索引为一个 _隐藏的独立文档_ ,举例如下:

```json
{ 
  "comments.name":    [ john, smith ],
  "comments.comment": [ article, great ],
  "comments.age":     [ 28 ],
  "comments.stars":   [ 4 ],
  "comments.date":    [ 2014-09-01 ]
}
{ 
  "comments.name":    [ alice, white ],
  "comments.comment": [ like, more, please, this ],
  "comments.age":     [ 31 ],
  "comments.stars":   [ 5 ],
  "comments.date":    [ 2014-10-22 ]
}
{ 
  "title":            [ eggs, nest ],
  "body":             [ making, money, work, your ],
  "tags":             [ cash, shares ]
}
```
第一个 `嵌套文档`
第二个 `嵌套文档`
_根文档_ 或者也可称为父文档

在独立索引每一个嵌套对象后,对象中每个字段的相关性得以保留。我们查询时,也仅仅返回那些真正符合条件的文档。

不仅如此,由于嵌套文档直接存储在文档内部,查询时嵌套文档和根文档联合成本很低,速度和单独存储几乎一样。

嵌套文档是隐藏存储的,我们不能直接获取。如果要增删改一个嵌套对象,我们必须把整个文档重新索引才可以。值得注意的是,查询的时候返回的是整个文档,而不是嵌套文档本身。

# 嵌套对象映射

设置一个字段为 `nested` 很简单 —  你只需要将字段类型 `object` 替换为 `nested` 即可：

```json
PUT /my_index
{
  "mappings": {
    "blogpost": {
      "properties": {
        "comments": {
          "type": "nested",
          "properties": {
            "name":    { "type": "string"  },
            "comment": { "type": "string"  },
            "age":     { "type": "short"   },
            "stars":   { "type": "short"   },
            "date":    { "type": "date"    }
          }
        }
      }
    }
  }
}
```

`nested` 字段类型的设置参数与 `object` 相同。

这就是需要设置的一切。至此，所有 `comments` 对象会被索引在独立的嵌套文档中。可以查看 [`nested` 类型参考文档](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/nested.html) 获取更多详细信息。

# 嵌套对象查询

由于嵌套对象 被索引在独立隐藏的文档中，我们无法直接查询它们。 相应地，我们必须使用 [`nested` 查询](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-nested-query.html) 去获取它们：

```json
GET /my_index/blogpost/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "eggs" 
          }
        },
        {
          "nested": {
            "path": "comments", 
            "query": {
              "bool": {
                "must": [
                  {
                    "match": {
                      "comments.name": "john"
                    }
                  },
                  {
                    "match": {
                      "comments.age": 28
                    }
                  }
                ]
              }
            }
          }
        }
      ]
}}}
```

`title` 子句是查询根文档的。
`nested` 子句作用于嵌套字段 `comments` 。在此查询中，既不能查询根文档字段，也不能查询其他嵌套文档。
`comments.name` 和 `comments.age` 子句操作在同一个嵌套文档中。

`nested` 字段可以包含其他的 `nested` 字段。同样地，`nested` 查询也可以包含其他的 `nested` 查询。而嵌套的层次会按照你所期待的被应用。

`nested` 查询肯定可以匹配到多个嵌套的文档。每一个匹配的嵌套文档都有自己的相关度得分，但是这众多的分数最终需要汇聚为可供根文档使用的一个分数。

默认情况下，根文档的分数是这些嵌套文档分数的平均值。可以通过设置 score_mode 参数来控制这个得分策略，相关策略有 `avg` (平均值), `max` (最大值), `sum` (加和) 和 `none` (直接返回 `1.0` 常数值分数)。

```json
GET /my_index/blogpost/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "eggs"
          }
        },
        {
          "nested": {
            "path": "comments",
            "score_mode": "max",
            "query": {
              "bool": {
                "must": [
                  {
                    "match": {
                      "comments.name": "john"
                    }
                  },
                  {
                    "match": {
                      "comments.age": 28
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```

返回最优匹配嵌套文档的 `_score` 给根文档使用。

如果 `nested` 查询放在一个布尔查询的 `filter` 子句中，其表现就像一个 `nested` 查询，只是 `score_mode` 参数不再生效。因为它被用于不打分的查询中 — 只是符合或不符合条件，不必打分 — 那么 `score_mode` 就没有任何意义，因为根本就没有要打分的地方。

# 使用嵌套字段排序

尽管嵌套字段的值存储于独立的嵌套文档中，但依然有方法按照嵌套字段的值排序。 让我们添加另一个记录，以使得结果更有意思：

```json
PUT /my_index/blogpost/2
{
  "title": "Investment secrets",
  "body":  "What they don't tell you ...",
  "tags":  [ "shares", "equities" ],
  "comments": [
    {
      "name":    "Mary Brown",
      "comment": "Lies, lies, lies",
      "age":     42,
      "stars":   1,
      "date":    "2014-10-18"
    },
    {
      "name":    "John Smith",
      "comment": "You're making it up!",
      "age":     28,
      "stars":   2,
      "date":    "2014-10-16"
    }
  ]
}
```

假如我们想要查询在10月份收到评论的博客文章，并且按照 `stars` 数的最小值来由小到大排序，那么查询语句如下：

```go
GET /_search
{
  "query": {
    "nested": { 
      "path": "comments",
      "filter": {
        "range": {
          "comments.date": {
            "gte": "2014-10-01",
            "lt":  "2014-11-01"
          }
        }
      }
    }
  },
  "sort": {
    "comments.stars": { 
      "order": "asc",   
      "mode":  "min",   
      "nested_path": "comments",
      "nested_filter": {
        "range": {
          "comments.date": {
            "gte": "2014-10-01",
            "lt":  "2014-11-01"
          }
        }
      }
    }
  }
}
```

此处的 `nested` 查询将结果限定为在10月份收到过评论的博客文章。

结果按照匹配的评论中 `comment.stars` 字段的最小值 (`min`) 来由小到大 (`asc`) 排序。

排序子句中的 `nested_path` 和 `nested_filter` 和 `query` 子句中的 `nested` 查询相同，原因在下面有解释。

我们为什么要用 nested_path 和 nested_filter 重复查询条件呢？原因在于，排序发生在查询执行之后。 查询条件限定了在10月份收到评论的博客文档，但返回的是博客文档。如果我们不在排序子句中加入 `nested_filter` ， 那么我们对博客文档的排序将基于博客文档的所有评论，而不是仅仅在10月份接收到的评论。

# 嵌套聚合

在查询的时候，我们使用 `nested` 查询 就可以获取嵌套对象的信息。同理， `nested` 聚合允许我们对嵌套对象里的字段进行聚合操作。

```go
GET /my_index/blogpost/_search
{
  "size" : 0,
  "aggs": {
    "comments": {
      "nested": {
        "path": "comments"
      },
      "aggs": {
        "by_month": {
          "date_histogram": { 
            "field":    "comments.date",
            "interval": "month",
            "format":   "yyyy-MM"
          },
          "aggs": {
            "avg_stars": {
              "avg": {
                "field": "comments.stars"
              }
            }
          }
        }
      }
    }
  }
}
```
`nested` 聚合 “进入” 嵌套的 `comments` 对象。
comment对象根据 comments.date 字段的月份值被分到不同的桶。
计算每个桶内star的平均数量。

从下面的结果可以看出聚合是在嵌套文档层面进行的：

```go
...
"aggregations": {
  "comments": {
     "doc_count": 4, 
     "by_month": {
        "buckets": [
           {
              "key_as_string": "2014-09",
              "key": 1409529600000,
              "doc_count": 1, 
              "avg_stars": {
                 "value": 4
              }
           },
           {
              "key_as_string": "2014-10",
              "key": 1412121600000,
              "doc_count": 3,
              "avg_stars": {
                 "value": 2.6666666666666665
              }
           }
        ]
     }
  }
}
...
```

总共有4个 `comments` 对象 ：1个对象在9月的桶里，3个对象在10月的桶里。

### 逆向嵌套聚合

`nested` 聚合 只能对嵌套文档的字段进行操作。 根文档或者其他嵌套文档的字段对它是不可见的。 然而，通过 `reverse_nested` 聚合，我们可以 _走出_ 嵌套层级，回到父级文档进行操作。

例如，我们要基于评论者的年龄找出评论者感兴趣 `tags` 的分布。 `comment.age` 是一个嵌套字段，但 `tags` 在根文档中：

```json
GET /my_index/blogpost/_search
{
  "size" : 0,
  "aggs": {
    "comments": {
      "nested": {
        "path": "comments"
      },
      "aggs": {
        "age_group": {
          "histogram": { 
            "field":    "comments.age",
            "interval": 10
          },
          "aggs": {
            "blogposts": {
              "reverse_nested": {}, 
              "aggs": {
                "tags": {
                  "terms": {
                    "field": "tags"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

`nested` 聚合进入 `comments` 对象。
`histogram` 聚合基于 `comments.age` 做分组，每10年一个分组。
`reverse_nested` 聚合退回根文档。
`terms` 聚合计算每个分组年龄段的评论者最常用的标签词。

简略结果如下所示：

```json
..
"aggregations": {
  "comments": {
     "doc_count": 4, 
     "age_group": {
        "buckets": [
           {
              "key": 20, 
              "doc_count": 2, 
              "blogposts": {
                 "doc_count": 2, 
                 "tags": {
                    "doc_count_error_upper_bound": 0,
                    "buckets": [
                       { "key": "shares",   "doc_count": 2 },
                       { "key": "cash",     "doc_count": 1 },
                       { "key": "equities", "doc_count": 1 }
                    ]
                 }
              }
           },
...
```

一共有4条评论。
在20岁到30岁之间总共有两条评论。
这些评论包含在两篇博客文章中。

在这些博客文章中最热门的标签是 `shares`、 `cash`、`equities`。

###  嵌套对象的使用时机

嵌套对象 在只有一个主要实体时非常有用，这个主要实体包含有限个紧密关联但又不是很重要的实体，例如我们的 `blogpost` 对象包含评论对象。 在基于评论的内容查找博客文章时， `nested` 查询有很大的用处，并且可以提供更快的查询效率。

嵌套模型的缺点如下：

-   当对嵌套文档做增加、修改或者删除时，整个文档都要重新被索引。嵌套文档越多，这带来的成本就越大。
-   查询结果返回的是整个文档，而不仅仅是匹配的嵌套文档。尽管目前有计划支持只返回根文档中最佳匹配的嵌套文档，但目前还不支持。

有时你需要在主文档和其关联实体之间做一个完整的隔离设计。这个隔离是由 _父子关联_ 提供的。

# 父-子关系文档

父-子关系文档 在实质上类似于 [nested model](https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-objects.html "嵌套对象") ：允许将一个对象实体和另外一个对象实体关联起来。而这两种类型的主要区别是：在 [`nested` objects](https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-objects.html "嵌套对象") 文档中，所有对象都是在同一个文档中，而在父-子关系文档中，父对象和子对象都是完全独立的文档。

父-子关系的主要作用是允许把一个 type 的文档和另外一个 type 的文档关联起来，构成一对多的关系：一个父文档可以对应多个子文档 。与 [`nested` objects](https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-objects.html "嵌套对象") 相比，父-子关系的主要优势有：

-   更新父文档时，不会重新索引子文档。
-   创建，修改或删除子文档时，不会影响父文档或其他子文档。这一点在这种场景下尤其有用：子文档数量较多，并且子文档创建和修改的频率高时。
-   子文档可以作为搜索结果独立返回。

Elasticsearch 维护了一个父文档和子文档的映射关系，得益于这个映射，父-子文档关联查询操作非常快。但是这个映射也对父-子文档关系有个限制条件：父文档和其所有子文档，都必须要存储在同一个分片中。

父-子文档ID映射存储在 [Doc Values](https://www.elastic.co/guide/cn/elasticsearch/guide/current/docvalues.html "Doc Values") 中。当映射完全在内存中时， [Doc Values](https://www.elastic.co/guide/cn/elasticsearch/guide/current/docvalues.html "Doc Values") 提供对映射的快速处理能力，另一方面当映射非常大时，可以通过溢出到磁盘提供足够的扩展能力

## 父-子关系文档映射

建立父-子文档映射关系时只需要指定某一个文档 type 是另一个文档 type 的父亲。 该关系可以在如下两个时间点设置：1）创建索引时；2）在子文档 type 创建之前更新父文档的 mapping。

举例说明，有一个公司在多个城市有分公司，并且每一个分公司下面都有很多员工。有这样的需求：按照分公司、员工的维度去搜索，并且把员工和他们工作的分公司联系起来。针对该需求，用嵌套模型是无法实现的。当然，如果使用 [application-side-joins](https://www.elastic.co/guide/cn/elasticsearch/guide/current/application-joins.html "应用层联接") 或者 [data denormalization](https://www.elastic.co/guide/cn/elasticsearch/guide/current/denormalization.html "非规范化你的数据") 也是可以实现的，但是为了演示的目的，在这里我们使用父-子文档。

我们需要告诉Elasticsearch，在创建员工 `employee` 文档 type 时，指定分公司 `branch` 的文档 type 为其父亲。

```json
PUT /company
{
  "mappings": {
    "branch": {},
    "employee": {
      "_parent": {
        "type": "branch"
      }
    }
  }
}
```

`employee` 文档 是 `branch` 文档的子文档。

## 构建父-子文档索引

为父文档创建索引与为普通文档创建索引没有区别。父文档并不需要知道它有哪些子文档。

```json
POST /company/branch/_bulk
{ "index": { "_id": "london" }}
{ "name": "London Westminster", "city": "London", "country": "UK" }
{ "index": { "_id": "liverpool" }}
{ "name": "Liverpool Central", "city": "Liverpool", "country": "UK" }
{ "index": { "_id": "paris" }}
{ "name": "Champs Élysées", "city": "Paris", "country": "France" }
```

创建子文档时，用户必须要通过 `parent` 参数来指定该子文档的父文档 ID：

```json
PUT /company/employee/1?parent=london
{
  "name":  "Alice Smith",
  "dob":   "1970-10-24",
  "hobby": "hiking"
}
```
当前 `employee` 文档的父文档 ID 是 `london` 。

父文档 ID 有两个作用：创建了父文档和子文档之间的关系，并且保证了父文档和子文档都在同一个分片上。

在 [路由一个文档到一个分片中](https://www.elastic.co/guide/cn/elasticsearch/guide/current/routing-value.html "路由一个文档到一个分片中") 中，我们解释了 Elasticsearch 如何通过路由值来决定该文档属于哪一个分片，路由值默认为该文档的 `_id` 。分片路由的计算公式如下：

shard = hash(routing) % number_of_primary_shards

如果指定了父文档的 ID，那么就会使用父文档的 ID 进行路由，而不会使用当前文档 `_id` 。也就是说，如果父文档和子文档都使用相同的值进行路由，那么父文档和子文档都会确定分布在同一个分片上。

在执行单文档的请求时需要指定父文档的 ID，单文档请求包括：通过 `GET` 请求获取一个子文档；创建、更新或删除一个子文档。而执行搜索请求时是不需要指定父文档的ID，这是因为搜索请求是向一个索引中的所有分片发起请求，而单文档的操作是只会向存储该文档的分片发送请求。因此，如果操作单个子文档时不指定父文档的 ID，那么很有可能会把请求发送到错误的分片上。

父文档的 ID 应该在 `bulk` API 中指定

```json
POST /company/employee/_bulk
{ "index": { "_id": 2, "parent": "london" }}
{ "name": "Mark Thomas", "dob": "1982-05-16", "hobby": "diving" }
{ "index": { "_id": 3, "parent": "liverpool" }}
{ "name": "Barry Smith", "dob": "1979-04-01", "hobby": "hiking" }
{ "index": { "_id": 4, "parent": "paris" }}
{ "name": "Adrien Grand", "dob": "1987-05-11", "hobby": "horses" }
```

如果你想要改变一个子文档的 `parent` 值，仅通过更新这个子文档是不够的，因为新的父文档有可能在另外一个分片上。因此，你必须要先把子文档删除，然后再重新索引这个子文档。

## 通过子文档查询父文档

`has_child` 的查询和过滤可以通过子文档的内容来查询父文档。例如，我们根据如下查询，可查出所有80后员工所在的分公司：

```json
GET /company/branch/_search
{
  "query": {
    "has_child": {
      "type": "employee",
      "query": {
        "range": {
          "dob": {
            "gte": "1980-01-01"
          }
        }
      }
    }
  }
}
```

类似于 [`nested` query](https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-query.html "嵌套对象查询") ，`has_child` 查询可以匹配多个子文档，并且每一个子文档的评分都不同。但是由于每一个子文档都带有评分，这些评分如何规约成父文档的总得分取决于 `score_mode` 这个参数。该参数有多种取值策略：默认为 `none` ，会忽略子文档的评分，并且会给父文档评分设置为 `1.0` ； 除此以外还可以设置成 `avg` 、 `min` 、 `max` 和 `sum` 。

下面的查询将会同时返回 `london` 和 `liverpool` ，不过由于 `Alice Smith` 要比 `Barry Smith` 更加匹配查询条件，因此 `london` 会得到一个更高的评分。

```json
GET /company/branch/_search
{
  "query": {
    "has_child": {
      "type":       "employee",
      "score_mode": "max",
      "query": {
        "match": {
          "name": "Alice Smith"
        }
      }
    }
  }
}
```

`score_mode` 为默认的 `none` 时，会显著地比其模式要快，这是因为Elasticsearch不需要计算每一个子文档的评分。只有当你真正需要关心评分结果时，才需要为 `score_mode` 设值，例如设成 `avg` 、 `min` 、 `max` 或 `sum` 。

### min_children 和 max_children

`has_child` 的查询和过滤都可以接受这两个参数：`min_children` 和 `max_children` 。 使用这两个参数时，只有当子文档数量在指定范围内时，才会返回父文档。

如下查询只会返回至少有两个雇员的分公司：

```json
GET /company/branch/_search
{
  "query": {
    "has_child": {
      "type":         "employee",
      "min_children": 2, [](https://www.elastic.co/guide/cn/elasticsearch/guide/current/has-child.html#CO280-1)
      "query": {
        "match_all": {}
      }
    }
  }
}
```

至少有两个雇员的分公司才会符合查询条件。

带有 `min_children` 和 `max_children` 参数的 `has_child` 查询或过滤，和允许评分的 `has_child` 查询的性能非常接近。

**has_child Filter**

`has_child` 查询和过滤在运行机制上类似，区别是 `has_child` 过滤不支持 `score_mode` 参数。`has_child` 过滤仅用于筛选内容—​如内部的一个 `filtered` 查询—​和其他过滤行为类似：包含或者排除，但没有进行评分。

`has_child` 过滤的结果没有被缓存，但是 `has_child` 过滤内部的过滤方法适用于通常的缓存规则

## 通过父文档查询子文档

虽然 `nested` 查询只能返回最顶层的文档 ，但是父文档和子文档本身是彼此独立并且可被单独查询的。我们使用 `has_child` 语句可以基于子文档来查询父文档，使用 `has_parent` 语句可以基于父文档来查询子文档。

`has_parent` 和 `has_child` 非常相似，下面的查询将会返回所有在 UK 工作的雇员：

```json
GET /company/employee/_search
{
  "query": {
    "has_parent": {
      "type": "branch",
      "query": {
        "match": {
          "country": "UK"
        }
      }
    }
  }
}
```
返回父文档 `type` 是 `branch` 的所有子文档

`has_parent` 查询也支持 `score_mode` 这个参数，但是该参数只支持两种值： `none` （默认）和 `score` 。每个子文档都只有一个父文档，因此这里不存在将多个评分规约为一个的情况， `score_mode` 的取值仅为 `score` 和 `none` 。

**不带评分的 has_parent 查询**

当 `has_parent` 查询用于非评分模式（比如 filter 查询语句）时， `score_mode` 参数就不再起作用了。因为这种模式只是简单地包含或排除文档，没有评分，那么 `score_mode` 参数也就没有意义了

## 子文档聚合

在父-子文档中支持 [子文档聚合](http://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-children-aggregation.html)，这一点和 [嵌套聚合](https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-aggregation.html "嵌套聚合") 类似。但是，对于父文档的聚合查询是不支持的（和 `reverse_nested` 类似）。

我们通过下面的例子来演示按照国家维度查看最受雇员欢迎的业余爱好：

```json
GET /company/branch/_search
{
  "size" : 0,
  "aggs": {
    "country": {
      "terms": { 
        "field": "country"
      },
      "aggs": {
        "employees": {
          "children": { 
            "type": "employee"
          },
          "aggs": {
            "hobby": {
              "terms": { 
                "field": "hobby"
              }
            }
          }
        }
      }
    }
  }
}
```

`country` 是 `branch` 文档的一个字段。
子文档聚合查询通过 `employee` type 的子文档将其父文档聚合在一起。
`hobby` 是 `employee` 子文档的一个字段。

## 祖辈与孙辈关系

父子关系可以延展到更多代关系，比如生活中孙辈与祖辈的关系 — 唯一的要求是满足这些关系的文档必须在同一个分片上被索引。

让我们把上一个例子中的 `country` 类型设定为 `branch` 类型的父辈：

```json
PUT /company
{
  "mappings": {
    "country": {},
    "branch": {
      "_parent": {
        "type": "country"
      }
    },
    "employee": {
      "_parent": {
        "type": "branch"
      }
    }
  }
}
```

`branch` 是 `country` 的子辈。
`employee` 是 `branch` 的子辈。

country 和 branch 之间是一层简单的父子关系，所以我们的 [操作步骤](https://www.elastic.co/guide/cn/elasticsearch/guide/current/indexing-parent-child.html "构建父-子文档索引") 与之前保持一致：

```json
POST /company/country/_bulk
{ "index": { "_id": "uk" }}
{ "name": "UK" }
{ "index": { "_id": "france" }}
{ "name": "France" }

POST /company/branch/_bulk
{ "index": { "_id": "london", "parent": "uk" }}
{ "name": "London Westmintster" }
{ "index": { "_id": "liverpool", "parent": "uk" }}
{ "name": "Liverpool Central" }
{ "index": { "_id": "paris", "parent": "france" }}
{ "name": "Champs Élysées" }
```

`parent` ID 使得每一个 `branch` 文档被路由到与其父文档 `country` 相同的分片上进行操作。然而，当我们使用相同的方法来操作 `employee` 这个孙辈文档时，会发生什么呢？

```json
PUT /company/employee/1?parent=london
{
  "name":  "Alice Smith",
  "dob":   "1970-10-24",
  "hobby": "hiking"
}
```

employee 文档的路由依赖其父文档 ID — 也就是 `london` — 但是 `london` 文档的路由却依赖 _其本身的_ 父文档 ID — 也就是 `uk` 。此种情况下，孙辈文档很有可能最终和父辈、祖辈文档不在同一分片上，导致不满足祖辈和孙辈文档必须在同一个分片上被索引的要求。

解决方案是添加一个额外的 `routing` 参数，将其设置为祖辈的文档 ID ，以此来保证三代文档路由到同一个分片上。索引请求如下所示：

```json
PUT /company/employee/1?parent=london&routing=uk
{
  "name":  "Alice Smith",
  "dob":   "1970-10-24",
  "hobby": "hiking"
}
```
`routing` 的值会取代 `parent` 的值作为路由选择。

`parent` 参数的值仍然可以标识 employee 文档与其父文档的关系，但是 `routing` 参数保证该文档被存储到其父辈和祖辈的分片上。`routing` 值在所有的文档请求中都要添加。

联合多代文档进行查询和聚合是可行的，只需要一代代的进行设定即可。例如，我们要找到哪些国家的雇员喜欢远足旅行，此时只需要联合 country 和 branch，以及 branch 和 employee：

```json
GET /company/country/_search
{
  "query": {
    "has_child": {
      "type": "branch",
      "query": {
        "has_child": {
          "type": "employee",
          "query": {
            "match": {
              "hobby": "hiking"
            }
          }
        }
      }
    }
  }
}
```
##   实际使用中的一些建议

当文档索引性能远比查询性能重要的时候，父子关系是非常有用的，但是它也是有巨大代价的。其查询速度会比同等的嵌套查询慢5到10倍!

###  全局序号和延迟

父子关系使用了[全局序数](https://www.elastic.co/guide/cn/elasticsearch/guide/current/preload-fielddata.html#global-ordinals "全局序号（Global Ordinals）") 来加速文档间的联合。不管父子关系映射是否使用了内存缓存或基于硬盘的 doc values，当索引变更时，全局序数要重建。

一个分片中父文档越多，那么全局序数的重建就需要更多的时间。父子关系更适合于父文档少、子文档多的情况。

全局序数默认情况下是延迟构建的：在refresh后的第一个父子查询会触发全局序数的构建。而这个构建会导致用户使用时感受到明显的迟缓。你可以使用[全局序数预加载](https://www.elastic.co/guide/cn/elasticsearch/guide/current/preload-fielddata.html#eager-global-ordinals "预构建全局序号（Eager global ordinals）") 来将全局序数构建的开销由query阶段转移到refresh阶段，设置如下：

```json
PUT /company
{
  "mappings": {
    "branch": {},
    "employee": {
      "_parent": {
        "type": "branch",
        "fielddata": {
          "loading": "eager_global_ordinals"
        }
      }
    }
  }
}
```
在一个新的段可搜索前，`_parent` 字段的全局序数会被构建。

当父文档过多时，全局序数的构建会耗费很多时间。此时可以通过增加 `refresh_interval` 来减少 refresh 的次数，延长全局序数的有效时间，这也很大程度上减小了全局序数每秒重建的cpu消耗。

###  多代使用和结语

多代文档的联合查询(查看 [祖辈与孙辈关系](https://www.elastic.co/guide/cn/elasticsearch/guide/current/grandparents.html "祖辈与孙辈关系"))虽然看起来很吸引人，但必须考虑如下的代价：

-   联合越多，性能越差。
-   每一代的父文档都要将其字符串类型的 `_id` 字段存储在内存中，这会占用大量内存。

当你考虑父子关系是否适合你现有关系模型时，请考虑下面这些建议：

-   尽量少地使用父子关系，仅在子文档远多于父文档时使用。
-   避免在一个查询中使用多个父子联合语句。
-   在 has_child 查询中使用 filter 上下文，或者设置 score_mode 为 none 来避免计算文档得分。
-   保证父 IDs 尽量短，以便在 doc values 中更好地压缩，被临时载入时占用更少的内存。

_最重要的是:_ 先考虑下我们之前讨论过的其他方式来达到父子关系的效果。

# 扩容设计
分片并不是没有代价的。记住：

-   一个分片的底层即为一个 Lucene 索引，会消耗一定文件句柄、内存、以及 CPU 运转。
-   每一个搜索请求都需要命中索引中的每一个分片，如果每一个分片都处于不同的节点还好， 但如果多个分片都需要在同一个节点上竞争使用相同的资源就有些糟糕了。
-   用于计算相关度的词项统计信息是基于分片的。如果有许多分片，每一个都只有很少的数据会导致很低的相关度。