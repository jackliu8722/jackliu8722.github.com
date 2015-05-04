---
layout: post
title: "Cassandra架构与源码分析(一)"
date: 2014-03-18 23:38
comments: true
categories: 分布式{distributed}
---
###Cassandra是什么
随着互联网的蓬勃发展，NoSQL数据库风生水起，涌现出了非常多的非关系数据库，比较火的有MongoDB,HBase，Cassandra等，我们这里主要讲Cassandra，那它是怎么样的一个NoSQL数据库呢？

Cassandra是一套混合型的非关系KV数据库系统，类似于Google的BigTable。大家应该知道Amazon的Dynamo分布式系统吧(不了解的可以google一下)，Cassandra的功能比Dynamo更丰富，但支持度却不如文档存储系统MongoDB。Cassandra最初由Facebook开发，后来转变成了一个开源项目。它是一个网络社交云计算方面理想的数据库。以Amazon专有的完全分布式的Dynamo为基础，并结合了Google的基于列族数据模型的Bigtable，是一个完全P2P去中心化的存储系统。