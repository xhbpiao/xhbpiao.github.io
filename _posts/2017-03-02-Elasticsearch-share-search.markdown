---
layout:     post
title:      "Elasticsearch 使用分享(下)"
subtitle:   "记录下ES的使用经验，帮助别人少走弯路"
date:       2017-03-02 19:00:00
author:     "孤鸿梦"
header-img: "img/post-es-share-bg.png"
catalog: true
tags:
    - 技术
    - Elasticsearch
    - 使用分享
    - ES 搜索
    - ES 索引
    - Elasticsearch index
    - ES 经验
    - Elasticsearch search
---

# 前言
> 这里不介绍如何安装，网上教程很多。不过建议参照[官网](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)进行安装。升级相关的问题可以参考[ES2.x升级到5.x注意事项](http://xhbpiao.com/2017/02/16/Elasticsearch-update/)。下面以ES来代替。 本篇只介绍搜索部分，[索引部分](http://xhbpiao.com/2017/03/02/Elasticsearch-share-index)


# 搜索介绍
> 只对常用的一些搜索条件进行介绍。更复杂的部分建议参考官方文档

- term
    > 精确匹配搜索，用于完全相同匹配。搜索效率最高，类似于mysql中的 = 条件查询,一般用于匹配数字，日期，枚举等类型结构，不建议用于文本匹配  
    因为在查询的时候，不知道改字段的索引类型，因此没办法进行分词匹配。  
    所以如果需要term 查询，该字段最好不进行分词  
    支持调整权重

    - 查询样例：
        ```json
        {
            "query": {
                "term" : { "user" : "Kimchy" } 
            }
        }
        ```
    - 调整权重查询：
        ```json
        {
          "term": {
            "status": {
              "value": "urgent",
              "boost": 2.0 
            }
          }
        }
        ```        
        > 其中 boost 就是权重（也就是相关度），默认为 1.0

        
- match
    > 匹配查询，该类型在接收到查询条件以后会进行分词解析，然后进行匹配查询。需要指定搜索域    
    支持文本、数字、日期等类型查询   
    支持模糊匹配，可指定匹配前缀长度  
    支持匹配模式：完全包含还是部分包含搜索条件，默认部分包含
    
    - 查询样例：
        ```json
        {
            "query": {
                "match" : {
                    "message" : "this is a test"
                }
            }
        }
        ```
        > 其中message 为搜索field
        
    - 包含模式样例：
        ```json
        {
            "query": {
                "match" : {
                    "message" : {
                        "query" : "this is a test",
                        "operator" : "and"
                    }
                }
            }
        }
        ```
        > 说明： 这种查询需要完全包含查询语句才有结果
        
- query_string
    > 非常常用的一种搜索方式，包含match的特性，又有扩展部分  
    可以不指定搜索域而进行所有字段匹配  
    可以指定默认搜索域（default_field）  
    可在搜索时指定分词器（analyzer）  
    支持默认搜索模式（default_operator，默认为or）  
    支持正则匹配搜索  
    支持多域搜索（fields）  
    
    - 普通样例：
        ```json
        {
            "query": {
                "query_string" : {
                    "default_field" : "content", 
                    "query" : "this AND that OR thus"
                }
            }
        }
        ```
        > 其中 default_field 可以不指定，则搜索所有域  
        AND 和 OR 为关键字 表示 this 和 that 或者 thus
        
    - 多域查询：
        ```json
        {
            "query": {
                "query_string" : {
                    "fields" : ["content", "name"],
                    "query" : "this AND that"
                }
            }
        }
        ```
        > 适用于明确知道哪些字段可能包含需要查询的信息  
        或者用于限制可查询的范围。 
        
        ```json
        {
            "query": {
                "query_string": {
                    "query": "(content:this OR name:this) AND (content:that OR name:that)"
                }
            }
        }
        ```
        > 另一种写法。 不过不建议这么写，因为容易引起歧义增加理解难度
        
    - 加权搜索：
        ```json
        {
            "query": {
                "query_string" : {
                    "fields" : ["content", "name^5"],
                    "query" : "this AND that OR thus"
                }
            }
        }
        ```
        > 调整name的权重，这样name字段匹配到的结果相关度就更高，搜索结果就更靠前  
        支持对搜索关键字进行加权，样式类似
    
    - 正则搜索：
        ```json
        {
           "query": {
                "query_string" : {
                    "fields" : ["city.*"],
                    "query" : "this AND that OR thus"
                }
            }
        }
        ```
        > 表示对city 下的所有域进行搜索
        
        ```json
        {
           "query": {
                "query_string" : {
                    "fields" : ["name"],
                    "query" : "qu?ck bro*"
                }
            }
        }
        ```
        > 其中 ? 表示单个任意字符 * 表示0到多个字符
        
        ```json
        {
           "query": {
                "query_string" : {
                    "fields" : ["name"],
                    "query" : "quikc~"
                }
            }
        }
        ```
        > 前缀模糊匹配查询 。 默认最多再匹配两个字符，但可以指定匹配长度。 如： quick~3

    - 范围查询：
        ```json 
        {
           "query": {
                "query_string" : {
                    "fields" : ["date"],
                    "query" : "[2012-01-01 TO 2012-12-31]"
                }
            }
        }
        ```
        > 表示日期 从 2012-01-01 到 2012-12-31 的数据 包含两端点，如果不包含用{}  
        适用于其他可比较的类型。  
        用 * 来表示不限制的一端  
        支持比较符号。如 >、=、< 等

        
- bool
    > 联合查询条件，用于根据需求进行组合的模式。

    - must:
        > 必须包含条件，可以是其他各种单类型搜索语句。如：term ，query_string, match 等
        
    - must_not:
        > 必须不包含条件。 
    
    - should：
        > 可以包含条件。 可以通过参数设置至少包含多少条件，默认为0
        
    - filter：
        > 过滤条件，类似于must，不过满足filter的搜索结果不计入分数计算，不干预搜索排序结果。  
        搜索结果可进行缓存，用于下次搜索
        
    - 使用样例：
        ```json
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
        > 说明： must，filter,must_not 均支持多条件模式，书写样式如 should
        
- aggs
    > 聚合查询，用于对搜索结果进行处理。  
    常用的分组计数（terms），去重计数（cardinality），时间分组计数（date_histogram）

    - 分组聚合（terms）
        ```json
        {
            "aggs" : {
                "genres" : {
                    "terms" : { 
                        "field" : "genre",
                        "size":5
                    }
                }
            }
        }
        ```
        > 根据genre field 的结果进行聚合数据。类似于MySQL中的group  
        size 用于指定返回结果个数，默认为10。根据自己的需求进行设置。（PS：之前size为0返回所有搜索结果，在5.0 以后取消了这个特性。必须指定一个正整数）
        干扰排序的话参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html#search-aggregations-bucket-terms-aggregation-order)
        
    - 去重计数（cardinality）    
        > 实现原理通过分配内存map的形式，将数据放入map中，最后计算容量。  
        如果分片的内存空间不足，则根据已经放入的数据和剩余容量对比，进行估算
        
        ```json
        {
           "aggs" : {
                "author_count" : {
                    "cardinality" : {
                        "field" : "author",
                        "precision_threshold":100
                    }
                }
            }
        }
        ```
        > 指定字段的数据只计算一次  
        一般用于和terms配合进行排重计算  
        precision_threshold 是统计的精确度，默认为3000，最大为40000。   
        一般情况下精确度是不需要设置的，如果太大会导致计算结果变慢  
        各种精确度错误率对比可以参考![image](http://olgd3m0ha.bkt.clouddn.com/cardinality_error.png)  
        具体说明信息见[官方文档]（https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics-cardinality-aggregation.html#_counts_are_approximate）
        
    - 时间分组计数（date_histogram）
        > 以时间为单位进行分组统计数据
        
        ```json
        {
            "aggs" : {
                "articles_over_time" : {
                    "date_histogram" : {
                        "field" : "date",
                        "interval" : "month",
                        "format" : "yyyy-MM-dd" 
                    }
                }
            }
        }
        ```
        > interval 支持 year, quarter, month, week, day, hour, minute, second，如果需要指定数字直接在单位前加即可。   
        同时支持缩写模式 y(years),M(months),w(weeks),d(days),h(hours),H(hours),m(minutes),s(second)  
        format 会对返回的结果（key_as_string）进行格式化。支持标准的时间格式
    
    - 嵌套聚合
        ```json
        {
            "aggs" : {
                "articles_over_time" : {
                    "date_histogram" : {
                        "field" : "date",
                        "interval" : "month",
                        "format" : "yyyy-MM-dd" 
                    },
                    "aggs":{
                        "author_count" : {
                            "cardinality" : {
                                "field" : "author",
                                "precision_threshold":100
                            }
                        },
                        "genres" : {
                            "terms" : { 
                                "field" : "genre",
                                "size":5
                            }
                        }
                    }
                }
            }
        }
        ```
        > 备注：除了去重聚合下不能嵌套子聚合，其他基本都可以

- parent-child
    > 联合搜索。适用于一对多的数据结构

    - 搜索样例：
        ```json
        {
            "query": {
                "has_child" : {
                    "type" : "blog_tag",
                    "score_mode" : "min",
                    "min_children": 2, 
                    "max_children": 10, 
                    "query" : {
                        "term" : {
                            "tag" : "something"
                        }
                    }
                }
            }
        }
        ```
        > socre_mode 可选参数。默认是max排序。  
        min_children/max_children 可选参数。用于指定至少匹配的个数。只有满足这个条件，才会返回匹配结果。 
