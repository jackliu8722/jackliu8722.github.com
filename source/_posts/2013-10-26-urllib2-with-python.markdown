---
layout: post
title: "如何使用urllib2获取网络资源"
date: 2013-10-26 17:00
comments: true
categories: Python{Python}
---
</br>
<b>注意</b>:本文翻译自Python官方文档，Python的版本为3.3.2

###1 简介
urllib.request是Python的一个模块，它可以通过URL（统一资源定位）获取网络上的信息。该模块提供了一个非常简单的接口函数urlopen,能够使用不同的协议获取URL指定的资源。它还提供了一个稍微复杂的接口处理常见的情况，比如基本的身份验证、cookies、代理等等。

urllib通过相关的网络协议（例如，FTP,HTTP）可以支持许多URL schemes（由URL字符串中的":"符号之前的字符串标识，例如，ftp是URL"ftp://python.org/"的schemes）.本教程重点介绍最常用的协议——HTTP。

对于简单的应用中，urlopen的使用是非常简单的，但是，当你打开HTTP协议的网址时，只要遇到一些错误或者不可处理的情况，就需要对HTTP协议有一定的理解。HTTP最全面最权威的参考文档是RFC 2616，这是一个技术文档，而且不易于阅读。此处的使用文档的目的是介绍urlib的使用，关于HTTP更多的细节可以参考RFC 2616文档，它的作用不是取代urllib.request文档，而是作为urllib.request的一个补充知识。

###2 获取网络资源
使用urllib.request的最简单的方法如下：

{% codeblock lang:python%}
	
	import urllib.request
	response = url lib.request.urlopen('http://python.org/')
	html = response.read()
{% endcodeblock%}

如果想通过URL检索资源，并将结果存储在一个临时位置，可以通过urlretrieve函数到达要求。
{% codeblock lang:python%}
	
	import urllib.request
	local_filename,headers = url lib.request.urlretrieve('http://python.org/')
	html = open(local_filename)
{% endcodeblock%}
urllib的使用就如以上的方法那样简单（注意，除了使用http,我们也可以使用ftp协议，file等等。），然而本教程的目的是介绍更复杂的HTTP协议。

HTTP协议包括请求和响应，客户端发出请求，而服务器响应请求。urllib.request代表了HTTP的请求对象，表示我们创建的服务请求。在最简单的形式中，首先创建一个Request对象，指定需要获取资源的URL，然后调用urlopen返回一个针对请求的响应对象，这处响应对象类似文件对象，也就是说，可以调用read函数获取响应的内容。