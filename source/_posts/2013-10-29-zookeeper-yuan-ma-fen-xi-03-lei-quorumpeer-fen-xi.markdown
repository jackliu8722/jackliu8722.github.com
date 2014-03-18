---
layout: post
title: "zookeeper源码分析(三)——类QuorumPeer的分析"
date: 2013-10-29 23:12
comments: true
categories: 分布式{distributed}
---
在Zookeeper中，类QuoromPeer处在非常重要的地位，它管理了quorum协议，并且包含以下三种状态：

1. Leader选择：每个server都将选出一个leader（最初建议server自己为leader）。
2. Follower：处于该状态的server将与leader进行同步，并且复制任务事务。
3. Leader：处于leader状态的server处理请求，并转发给Follower。大部分Follower在接受请求之前必须将请求输出到日志中。
