---
layout: post
title: "Merkle Tree"
date: 2014-03-18 11:32
comments: true
categories: 分布式{distributed}
---
###什么是Merkle Tree
Merkle Tree是一种树，并且该树所有节点存储的都是hash值，因此网上大都把Merkle Tree又称为Merkle Hash Tree。Merkle Tree具有以下的特点：

1. Merkle Tree是一种树，并且具有树的所有特点，可以是二叉树也可是多叉树；
2. Merkle Tree的叶子节点存储的value由设计者指定，可以存储具体的数据，也可以存储数据的hash值；
3. Merkle Tree非叶子的value存储的其所有子节点的hash值，计算hash值的算法由设计者制定；

下图是Merkle Tree的一个例子
{% img  center /images/merkle-tree.png %}

根据上图的一棵Merkle Tree，可以得到如下的结果：

1. 该Merkle Tree是一棵二叉树；
2. 