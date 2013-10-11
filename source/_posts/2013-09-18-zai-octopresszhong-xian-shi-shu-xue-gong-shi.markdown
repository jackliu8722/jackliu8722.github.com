---
layout: post
title: "在Octopress中显示数学公式"
date: 2013-09-18 23:17
comments: true
categories: 其它{others}
---
由于个人对机器学习比较感兴趣，而机器学习的学习过程中会遇到非常多的数学公式，因此，写博客的过程中离不开公式的撰写。

平时写博客是在Mou下进行的，而Mou是支持[**LaTeX**](http://www.latex-project.org/)的，因此，通过Mou写的公式在博客未发布前是可能预览的。但是博客发布，显示的却不是公式，而是一串字符串，在网上找了下，原来Octopress要支持数学公式是需要安装插件的，在网上找了找记下配置过程和大家分享。

###安装kramdown
kramdown是什么，我这里就不多介绍了，有兴趣大家可以点击[**这里**](http://kramdown.rubyforge.org/)。
通过以下命令安装kramdown包
{% codeblock %}
gem install kramdown
{% endcodeblock%}

待安装完成之后进入 *OCTOPRESS_PATH/source/_includes/custom/* 目录，*OCTOPRESS_PATH*是Octopress的根目录。在该目录下找到head.html文件，在该文件中添加以下内容：
{% codeblock lang:javascript %}
<!-- mathjax config similar to math.stackexchange -->

<script type="text/x-mathjax-config">
  MathJax.Hub.Config({
    tex2jax: {
      inlineMath: [ ['$$$','$$$'], ["\\(","\\)"] ],
      processEscapes: true
    }
  });
</script>

<script type="text/x-mathjax-config">
    MathJax.Hub.Config({
      tex2jax: {
        skipTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code']
      }
    });
</script>

<script type="text/x-mathjax-config">
    MathJax.Hub.Queue(function() {
        var all = MathJax.Hub.getAllJax(), i;
        for(i=0; i < all.length; i += 1) {
            all[i].SourceElement().parentNode.className += ' has-jax';
        }
    });
</script>

<script type="text/javascript"
   src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>
{% endcodeblock %}

配置完成之后，就可以在Mou中使用LaTeX来撰写数学公式，并且发布后能够正常显示。