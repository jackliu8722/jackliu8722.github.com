---
layout: post
title: "Innodb源码学习(1)--Mysql环境的搭建"
date: 2015-11-14 16:39
comments: true
categories: 存储{db}
---

最近一直在研究Innodb的源码，主要的目的为了了解一种存储系统的实现，同时希望在工作当中能够学以致用。Innodb已经成为了Mysql的默认存储引擎，而且市面上各大互联网公司都在使用该存储引擎，因此花点时间研究一下还是非常有必要的。

要想深入的理解Innodb的精髓，在学习的过程中不得不对源码进行修改并编译调试，因此首先就需要编译安装mysql,下面介绍一下mysql编译安装的过程。

##源码下载
首先从mysql的官网上<http://dev.mysql.com/downloads>下载相应版本的源码，本文下载的源码包 mysql-5.6.20.tar.gz

##解压
假设我们下载好的源码在~/workspace/目录下，执行以下操作将源码解压
{% codeblock lang:sh%}
jackliu8722:~$ cd workspace
jackliu8722:~/workspace$ tar -zxvf mysql-5.6.20.tar.gz
{% endcodeblock%}

##编译
进入mysql源码目录并编译
{% codeblock lang:sh%}
jackliu8722:~/workspace$ cd mysql-5.6.20
jackliu8722:~/workspce/mysql-5.6.20$ cmake . \
        -DCMAKE_INSTALL_PREFIX=/home/jackliu8722/workspace/mysql-5.6.20-bin \        -DMYSQL_DATADIR=/home/jackliu8722/workspace/mysql-5.6.20-data \        -DMYSQL_UNIX_ADDR=/home/jackliu8722/workspace/mysql-5.6.20-bin/mysqld-5.6.20.sock \        -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_DEBUG=1
{% endcodeblock%}
上面操作完成之后执行以下操作进行编译安装
{% codeblock lang:sh%}
jackliu8722:~/workspace/mysql-5.6.20$ make && make install
{% endcodeblock %}
等待一会儿如果没有遇到什么错误，则mysql就编译完成了，并且安装到了/home/jackliu8722/workspace/mysql-5.6.20-bin/ 目录下。如果遇到什么依赖错误，自行安装重试即可，这里就不多介绍了。

##启动mysql服务
编译安装完成之后，在启动mysql服务之前，需要初始化数据库，按以下操作即可。
{% codeblock lang:sh %}
jackliu8722:~/workspace/mysql-5.6.20$ cd ../mysql-5.6.20-bin
jackliu8722:~/workspace/mysql-5.6.20-bin$ ./scripts/mysql_install_db --user=mysql
{% endcodeblock %}

然后启动mysql服务
{% codeblock lang:sh %}
jackliu8722:~/workspace/mysql-5.6.20-bin$ support-files/mysql.server start
{% endcodeblock %}

通过以上几步操作mysql的编译安装就完成了，为以验证是否安装成功，可以通过mysql客户端连接进行验证，如果能够连接成功说明安装已经成功了，就可以进行后续的源码学习了。




