---
layout:     post
title:      "Elasticsearch 2.X 升级到 5.X 注意事项"
subtitle:   "记录下升级中踩的坑"
date:       2017-02-16 14:00:00
author:     "孤鸿梦"
header-img: "img/post-bg-es-update.png"
catalog: true
tags:
    - 技术
    - Elasticsearch
    - 升级问题
---

# 前言
> [Elasticsearch](https://www.elastic.co/products/elasticsearch)是一个现在比较流行的实时是搜索引擎，它基于lunceue实现。 主要特点：**分布式的实时文件存储，每个字段都被索引并可被搜索；分布式的实时分析搜索引擎；
可以扩展到上百台服务器，处理PB级结构化或非结构化数据**


# 升级变更部分
### 配置参数
> 如果你是首次安装可能注意不到这些变化，如果是升级并想使用原来的配置，需要注意以下几点
1. bootstrap.mlockall 变为 bootstrap.memory_lock 
2. 由于5.1.2 （官方说5.1.2修复这个问题。经测试5.2 才真正修复）之前的bug 需要强制使用netty3
    1. transport.type: netty3
    2. http.type: netty3

### 启动部分
- 系统最低要求java 版本为1.8
- 默认内存改为2G 
- 启动传输参数的-D被-E替代
- ES_HEAP_SIZE 启动参数取消
    - 如：ES_JAVA_OPTS="-Xms2g -Xmx2g" ./bin/elasticsearch  (测试并没有生效)
    - 通过更改conf 下面的 jvm.options  进行更改内存占用（测试可用）
-  es.default.path.home ,es.default.path.conf,discovery.zen.ping.multicast.enabled 等参数被取消
- 参数设置不允许设置为空
- 因部分机器安全设置 ES 申请内存会受限，需要在 /etc/security/limits.conf 文件下 添加 
    - elasticsearch soft memlock unlimited
    - elasticsearch hard memlock unlimited

### 索引结构
> 如果你没使用模板的话这些变动你可能是无感知的
- string类型消失，由text和keyword来替换
  - text：分词并索引
  - keyword：不分词索引，主要用于聚合（aggs）和排序

### 搜索部分
- 限制搜索返回结果的'fields'由'_source'代替

### 插件部分
> 出于安全的考虑，ES在5.X以后将插件进行了限制，不再支持非官方的扩展

- elasticsarch-head
    >如果你偶尔会在界面化操纵数据查询的化你应该会使用这个插件 ，但是5.X已经不能像2.X那样进行安装了。新的是以独立服务的方式进行安装运行
    
    1. 首先更新 npm 源 
        - sudo npm install -g cnpm --registry=https://registry.npm.taobao.org
        - ps：如果 安装还是失败 ，升级 npm 重新更新源
    2. 如果没有 grunt ，进行安装 sudo cnpm install -g grunt-cli
    3. git clone git://github.com/mobz/elasticsearch-head.git
    4. cd elasticsearch-head
    5. cnpm install 
    6. nohup grunt server & 启动即可
    
    **备注:**
    - 由于在ES 5.X POST 和 GET 已经基本没什么区别了(都会解析body中的数据)。导致 elasticsearh-head 的GET 请求会出错（如果有body 数据则传输这部分数据，没有则传{}），所以如果需要GET 请求的话需要在浏览器中其他tag执行

- XPack 
    > ES官方出的多方位监控插件，功能很强大![image](http://olgd3m0ha.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-16%2014.56.11.png) 安装以后如果只是监控使用而不使用认证的话需要设置参数 xpack.security.enabled: false

    1. 在ES集群中的每个节点安装 x-pack
        - cd $ES_PATH
        - bin/elasticsearch-plugin install x-pack
    2. 在Kibana中安装 x-pack
        - cd $Kibana_Path
        - bin/kibana-plugin install x-pack
    3. 重启ES 集群
    4. 重启kibana 节点
    5. 在kibana 中 即可看到monitor界面
    

### kibana和logstash（只说升级部分，使用下次再聊）

- kibana 必须和 ES 同步升级（大版本，小版本可以不升级），否则可能导致不可用
- logstash 不建议升级，因为很多配置参数调整，升级代价有点高

### 错误现象和解决办法

> ![image](http://olgd3m0ha.bkt.clouddn.com/QQ%E5%9B%BE%E7%89%8720170216141221.png)
> 解决办法 参考配置参数 第2项 



