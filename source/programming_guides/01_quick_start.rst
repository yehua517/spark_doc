========================
快速入门
========================

* Spark Shell命令行操作
     * 基础操作
     * RDD扩展操作
     * 缓存(Caching)
* 内置应用
* 更多内容链接

简介
------------------------
这个教程提供了一个学习spark的快速介绍。我们将会通过spark的shell命令行介绍它的API(使用python或者scala)，然后演示如何在java,scala,python语言中编程，查看完整的参考请到这里：`编程指南 http://spark.apache.org/docs/latest/programming-guide.html`_

为了学习下面的操作，需要首先去官网下载spark安装包，<a>http://spark.apache.org/downloads.html</a>  。因为在下面的例子中我们不会使用hdfs，所以你可以去官网下载任意版本的spark安装包。



Spark Shell命令行操作
---------------------
基本操作
~~~~~~~~~
spark shell提供了一个简单的方法去学习API，以及一个强大的工具去解析数据。在scala或者python下都是可用的，在spark的指定目录下面运行即可。

scala::

	./bin/spark-sehll

python::

	./bin/pyspark

saprk中一个抽象的数据集合称为RDD(Resilient Distributed Dataset),RDDs可以从hadoop中的hdfs文件或者通过其他RDD转换得到。让我们通过spark源码目录中的README文件来创建一个新的RDD。

注意：在这里sc可以直接使用，spark shell会默认创建这个对象::

	>>> textFile = sc.textFile(“README.md")


RDDs有很多action操作<a>http://spark.apache.org/docs/latest/programming-guide.html#actions</a>，哪个返回值和transformation，哪个返回一个新的RDD，我们来尝试几个操作吧::

	>>> textFile.count() # 返回这个RDD中有多少个元素
	126

	>>> textFile.first() # 返回RDD中的第一个元素
	u'# Apache Spark'


现在让我们使执行一个transformation操作，我们将会使用filter 过滤算子返回一个包含之前RDD中满足条件数据的新RDD::

	>>> linesWithSpark = textFile.filter(lambda line: "Spark" in line)


我们可以在一块使用transformations和actions::

	>>> textFile.filter(lambda line: "Spark" in line).count() # 多少行包含saprk这个关键字?
	15

解释：transformation和action
------------------------------
RDD提供了两种类型的操作：transformation和action::

	其实，如果大家有hadoop基础，为了理解方便的话，可以这样理解
	hadoop中的mr计算框架中包含map操作和reduce操作，
	spark计算框架中包含transformation操作和action操作
	但是注意：前期为了好理解可以暂且这样理解，其实这个解释是不对的，这个等后期熟悉了之后就可以区分开了。

1，transformation是得到一个新的RDD，方式很多，比如从数据源生成一个新的RDD，从RDD生成一个新的RDD

2，action是得到一个值，或者一个结果（直接将RDD cache到内存中）
所有的transformation都是采用的懒策略，就是如果只是将transformation提交是不会执行计算的，计算只有在action被提交的时候才被触发
