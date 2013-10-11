---
layout: post
title: "Zookeeper-源码分析(一)——简介"
date: 2013-10-09 23:10
comments: true
categories: 分布式{distributed}
---
##Zookeeper是什么
Zookeeper是Hadoop的一个子项目，它是一个针对大型分布式系统的可靠协调系统，提供的功能包括：维护配置信息(maintaining configuration information)、名字服务(naming service)、分布式同步(distributed synchronization)、组服务(group services)等等，以上的服务经常应用在分布式系统当中。

Zookeeper的目标是封装好复杂并易出错的关键服务，将简单易用的接口和性能高效、功能稳定的系统提供给用户，从而降低用户实现分布式系统的难度。