---
layout: post
title: "Hello Octopress"
date: 2012-12-17 20:16
comments: true
categories: 
---
##引言
   以前光顾着学习了，不爱写博客，马上就要工作了，得养成写博客总结的习惯，从现在做起吧。Hello World大家都见过很多了，今天我来写个Hello Octopress，其实大家都明白，跟Hello World没啥区别，让我们开始吧。
##Hello Octopress
{% codeblock hello_octopress.py lang:python%}
class HelloOctopress:
	def __init__(self):
		self.msg = "Hello Octopress!"
	def print(self):
		for k in range(10):
			print self.msg
ho = HelloOctopress()
ho.print()
{% endcodeblock%}
##结论
哈哈，代码就几行，Hello Octopress完成！
期待自己写一篇精彩的文章！
Good luck!