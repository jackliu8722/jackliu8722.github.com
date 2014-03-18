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

{% codeblock lang:python%}
	
	import urllib.request
	req = urllib.request.Request('http://www.voidspace.org.uk')
	response = urllib.request.urllopen(req)
	the_page = response.read()
{% endcodeblock%}
需要注意的是，urllib.request使用相同的请求接口来处理所有的URL方案。例如，可以使用以下的方式使用FTP请求：

{% codeblock lang:python%}
	
	req = urllib.request.Request('ftp://example.com/')
{% endcodeblock%}

在处理HTTP的情况下，请求对象允许你可以做额外的两件事：首先，可以将数据发送给服务器。其次你可以将与数据有关的其它信息（元数据）或者请求自身发送给服务器，这些信息通过设置HTTP头进行传输。让我们依次介绍这些内容。

####2.1 数据
有时我们想向一个URL指定的地址发送数据（通常URL地址涉及一个CGI(公共网关接口)脚本或者其它的web应用程序）。在HTTP协议中，我们通常使用POST请求来完成这样的任务。当我们提交一个HTML表单时，浏览器通常使用POST请求，并非所有POST请求都来自表单。我们可以使用一个POST向我们的应用程序传输任意的数据。在HTML表单中，数据使用一种标准的方式进行编码，然后以参数的形式传给Request对象，在Python中，使用urllib.parse库来完成数据的编码。

{%codeblock lang:python%}

	import urllib.parse
	import urllib.request
	url = 'http://www.someserver.com/cgi-bin/register.cgi'
	values = {'name' : 'Michael Foord',
			   'location' : 'Northampton',
			   'language' : 'Python'}
	
	data = urllib.parse.urlencode(values)
	data = data.encode('utf-8')
	req = urllib.request.Request(url,data)
	response = url lib.request.urlopen(req)
	the_page = response.read()
{%endcodeblock%}

需要注意的是，有时候需要设置其它的编码方式（如上传文件的HTML表单——更多的介绍参见HTML规范中的表单提交）。

如果不传递参数data，则urllib使用GET请求。GTE请求与POST请求之间存在区别，其中一个区别是POST请求往往存在副作用，它可以以某种方式改变系统的状态。尽管HTTP协议的标准制定者清楚知道POST请求总是会产生副作用，GET请求永远不会引起副作用，没有什么能够使GET请求生产副作用，也没有什么能够使用HOST请求无副作用。数据也可以在HTTP GET请求中，通过URL编码的方式进行传递。通过以下方式实现：

{% codeblock lang:python%}

	import urllib.request
	import urllib.parse
	data = {}
	data['name'] = 'Somebody Here'
	data['location'] = 'Northhampton'
	data['language'] = 'Python'
	url_values = urllib.parse.urlencode(data)
	print(url_values)
	
	url = 'http://www.example.com/example.cgi'
	full_url=url + "?" + url_values
	data = urllib.request.urlopen(full_url)
{% endcodeblock%}

请注意，通过添加符号?并紧跟着编码好的值在其后的方式来创建完整的URL。

