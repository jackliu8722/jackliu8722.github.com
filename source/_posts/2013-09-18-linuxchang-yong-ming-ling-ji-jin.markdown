---
layout: post
title: "linux常用命令集锦"
date: 2013-09-18 22:54
comments: true
categories: Linux{linux}
---
##说明
本文主要把工作中和学习中遇到的一些常用的linux命令进行了整理，方便今后查阅，以节省时间，时间就是Money啊，哈哈……

##1. tail
###作用###

该命令从指定的行开始将文件写到标准输出。使用tail命令的**-f**选项可以方便的查阅正在改变的日志文件，tail -f filename会把filename里最尾部的内容显示在屏幕上，并且不但刷新，使你看到最新的文件内容。

###语法###

tail [  -f ] [  -c Number |  -n Number |  -m Number |  -b Number |  -k Number ] [ File ]
