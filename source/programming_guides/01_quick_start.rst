快速入门
=============

.. Tip:: 本文档基于spark2.1.1版本翻译，目前只翻译scala相关语法，不涉及python。

* Spark Shell命令行操作
    * 基础操作
    * RDD扩展操作
    * 缓存(Caching)
* 内置应用
* 更多内容链接

简介
------------------------
这个教程提供了一个学习spark的快速介绍。我们将会通过spark的shell命令行介绍它的API(使用python或者scala)，然后演示如何在java,scala,python语言中编程，查看完整的参考请到这里：`编程指南 <http://spark.apache.org/docs/latest/programming-guide.html>`_

为了学习下面的操作，需要首先去官网下载spark安装包，`官网下载地址 <http://spark.apache.org/downloads.html>`_ 。因为在下面的例子中我们不会使用hdfs，所以你可以去官网下载任意版本的spark安装包。

Spark Shell命令行操作
---------------------
