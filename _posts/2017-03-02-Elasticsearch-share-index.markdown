---
layout:     post
title:      "Elasticsearch 使用分享(上)"
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
> 这里不介绍如何安装，网上教程很多。不过建议参照[官网](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)进行安装。升级相关的问题可以参考[ES2.x升级到5.x注意事项](http://xhbpiao.com/2017/02/16/Elasticsearch-update/)。下面以ES来代替。 本篇只介绍索引部分，[搜索部分](http://xhbpiao.com/2017/03/02/Elasticsearch-share-search)

# ES 是什么
- 你可能经常听到ELK 它其实是ES,kibana,logstash的简称。而ES则是整个ELK生态环境的核心
- ES 是一个基于Apache Lucene(TM)的开源搜索引擎。无论在开源还是专有领域，Lucene可以被认为是迄今为止最先进、性能最好的、功能最全的搜索引擎库。
- 但是，Lucene只是一个库。想要使用它，你必须使用Java来作为开发语言并将其直接集成到你的应用中，更糟糕的是，Lucene非常复杂，你需要深入了解检索的相关知识来理解它是如何工作的。
- ES 也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单

# ES的主要特点

- 分布式的实时文件存储，每个字段都被索引并可被搜索
- 分布式的实时分析搜索引擎
- 可以扩展到上百台服务器，处理PB级结构化或非结构化数据
- 完善的生态环境：配套的数据收集、分析工具和详细的监控插件
 
# ES索引介绍
- index
    > 如果你用过数据库的话index就可以理解成库名

    - 是相似数据类型的聚合名称
    - 是搜索数据的入口
    
- type
    > 简单些可以理解为数据表。
    
    - 同类型数据的聚合名称
    - 同类型的数据入口
    
- _source
    > 数据表中的具体内容

- mapping
    > 数据表中的列属性聚合，默认情况下是注意不到这个东西的.  
    它描述了文档中可能包含的每个字段的属性;  
    标示字段是否需要被 Lucene索引,分词或储存等
    
    - 如何获取map

        - 获取所有索引map /_mapping 
        - 获取单个索引map /index/_mapping
        - 获取单个索引指定type的map /index/_mapping/type
        
    - map的结构
        ```json 
        {
            "bid": {
                "type": "text",
                "fields": {
                    "raw": {
                        "type": "keyword",
                        "ignore_above": 256
                    }
                }
            },
            "buffer": {
                "properties": {
                    "count": {
                        "type": "long"
                    },
                    "uid": {
                        "type": "long"
                    }
                }
            },
            "time": {
                "type": "date"
            }
        }
        ```
        > 说明:  
        1. 默认情况下string类型的数据自动识别为text模式  
        2. text后面的fields 是模版控制的结果，下面在介绍  
        3. 数字类型的会自动识别为long类型。如果以后的类型转变除非能转换成数字，否则会出错  
        4. 如果需要date类型，需要预指定，或者按照标准时间格式化之后会自动识别成date模式
        
- settings
    > 索引结构的设置配置  
    默认情况下通过elasticsearch.yml 配置即可。  
    为了能够更灵活的控制，依然支持接口来调整  
    
    * 获取settings模式 
        - 获取所有索引settings  /_mapping 
        - 获取单个索引settings  /index/_mapping
    
    * settings 格式
    ```json 
    {
        "index": {
            "creation_date": "1488124805703",
            "number_of_shards": "5",
            "number_of_replicas": "1",
            "uuid": "fULc_u_3T_KDGp5AgyCv8g",
            "version": {
                "created": "5010199"
            },
            "provided_name": "kafka-log-2017.02.27"
        }
    }
    ```  
    > 说明：  
    1. number_of_shards 表示索引分片个数  
    2. number_of_replicas 表示索引备份个数  
    3. 上面的信息里面只有number_of_replicas 是可以索引以后更改的，其他部分在创建的时候已经确定
    
- template 
    > 模版主要用于那些会自动创建新的索引，而且需要对索引结构进行调整，以便满足进行特殊处理的需求。 如：  
    1. 需要每天收集日志数据，同时需要对数据结构进行聚合处理  
    2. 需要完全匹配的需求如 term 等  
    3. 手动干预指定类型或字段的索引结构等
    
    - 样例：
    ```json
    {
    "template": "*",
    "settings": {
        "index": {
            "number_of_shards": "5",
            "number_of_replicas": "1"
        }
    },
    "mappings": {
        "_default_": {
            "dynamic_templates": [
                {
                    "strings": {
                        "mapping": {
                            "type": "text",
                            "fields": {
                                "raw": {
                                    "ignore_above": 256,
                                    "type": "keyword"
                                }
                            }
                        },
                        "match_mapping_type": "string",
                        "match":"*"
                    }
                }
            ]
        }
    }
    } 
    ```
    - **说明**：  
    1. template 是模板用于匹配的名字，支持前后缀模糊匹配（测试时中间匹配没有生效，建议前缀匹配）。当前设置为\* 表示匹配所有  
    2. settings 部分表示索引创建的默认结构  
    3. mappings 部分就是需要干扰索引结构的部分。 略过前两个关键字(\_default\_,dynamic_templates) 直接开始介绍里面的内容 
        - strings: 是模板名称，可根据实际情况自己指定
            - mapping: 映射关键字
                - type: 表示转换的类型：这里表示转换成 text 类型
                - fields: 表示隐藏字段 用于聚合（aggs）和 完全匹配搜索（term）
                    - raw: 隐藏字段名称
                        - ignore_above: 表示长度限制，超过256 将不进行添加改隐藏字段
                        - type: 隐藏字段类型 keyword （不进行分词索引）
            - match_mapping_type: 表示匹配数据类型：这里表示匹配 string 类型的数据
            - match: 表示匹配的字段 如果需要匹配所有的 设置为 '*' 就可以了。 如果需要对特殊字段处理，这里就填字段名称即可
    
- 联合索引parent-child
    > 适用于一主多从的数据结构。 如：大公司包含很多子公司的人员管理结构，子公司建立独立的索引结构，同时需要保留对应的关系。  
    当需要查询一个不确定在哪的用户信息时：  
    查询主公司，同时查询子公司下所有包换此人的详细信息

    - 说明：
        > 该类型对索引结构要求较高。
        
    - 设置样例 
        ```json
        {
            "mappings": {
                "my_parent": {},
                "my_child": {
                    "_parent": {
                        "type":        "my_parent" 
                    }
                }
            }
        }
        ```
        
    1. 在索引数据前需要建立父子索引结构
    2. 必须在相同的索引index 下
    3. 索引的type必须不相同
    4. 父类的type之前必须是不存在的
    5. 子类的数据必须和父节点的数据在同一个分片上，而子节点的数据是通过parent id进行route的。所以如果单独查询子节点数据需要指定和parent 相同的routing值
    
- 文件附件索引（attachment）
    > 基于[Tika](http://lucene.apache.org/tika/)实现，适用于福建类型的搜索，如pdf，word，excel 等.这种需求并不常见，如果不需要刻意忽略

    - 安装方式：
        sudo bin/elasticsearch-plugin install mapper-attachments
    
    - 索引结构设置：
        ```json
        {
            "mappings": {
                "person": {
                    "properties": {
                        "cv": { 
                            "type": "attachment"
                        }
                    }
                }
            }
        }
        ```
    
    - 索引数据方式：
        > 通过 base64-encoded 进行传输数据
    
        > POST /trying-out-mapper-attachments/person/1?refresh 
        
        ```json
        {
            "cv": "e1xydGYxXGFuc2kNCkxvcmVtIGlwc3VtIGRvbG9yIHNpdCBhbWV0DQpccGFyIH0="   
        }
        ```
        
    - 查询方式：
        > POST /trying-out-mapper-attachments/person/_search 
        
        ```json
        {   
            "query": {   
                "query_string": {   
                    "query": "ipsum"  
                }   
            }  
        }
        ```


