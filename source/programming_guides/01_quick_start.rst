快速入门
=============

.. Tip:: 本文档基于spark2.1.1版本翻译，目前只翻译scala相关语法，不涉及python。

* Spark Shell命令行操作
    * 基础操作
    * RDD扩展操作
    * 缓存(Caching)
* spark程序开发
* 更多内容链接

简介
------------------------
这个教程提供了一个学习spark的快速介绍。我们将会通过spark的shell命令行介绍它的API(使用python或者scala)，然后演示如何在java,scala,python语言中编程，查看完整的参考请到这里：`编程指南 <http://spark.apache.org/docs/latest/programming-guide.html>`_

为了学习下面的操作，需要首先去官网下载spark安装包，`官网下载地址 <http://spark.apache.org/downloads.html>`_ 。因为在下面的例子中我们不会使用hdfs，所以你可以去官网下载任意版本的spark安装包。

Spark Shell命令行操作
---------------------

基本操作
~~~~~~~~~

spark shell提供了一个简单的方法去学习API，以及一个强大的工具去解析数据。在scala
或者python下都是可用的，在spark的指定目录下面运行即可。

scala::

    ./bin/spark-sehll

saprk中一个抽象的数据集合称为RDD(Resilient Distributed Dataset),RDDs可以从hadoop中的hdfs文件或者通过其他RDD转换得到。让我们通过spark源码目录中的README文件来创建一个新的RDD。

.. Attention:: 注意:在这里sc可以直接使用，spark shell会默认创建这个对象

::

    scala> val textFile = sc.textFile("README.md")
    textFile: org.apache.spark.rdd.RDD[String] = README.md MapPartitionsRDD[1] at textFile at <console>:25


RDDs有很多 `action操作 <http://spark.apache.org/docs/latest/programming-guide.html\#actions>`_ ，哪个返回值和transformation，哪个返回一个新的RDD，我们来尝试几个操作吧:

::

    scala> textFile.count() // 返回这个RDD中有多少个元素
    res0: Long = 126 // 这个值可能会根据你使用的spark版本不同而不同，因为不同版本的README.md文件中的内容可能不同

    scala> textFile.first() // 返回RDD中的第一个元素
    res1: String = # Apache Spark


现在让我们使执行一个transformation操作，我们将会使用filter 过滤算子返回一个包含
之前RDD中满足条件数据的新RDD:

::

    scala> val linesWithSpark = textFile.filter(line => line.contains("Spark"))
    linesWithSpark: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[2] at filter at <console>:27

我们可以在一块使用transformations和actions:

::

    scala> textFile.filter(line => line.contains("Spark")).count() // 有多少行包含spark关键词
    res3: Long = 15

``解释：transformation和action``

RDD提供了两种类型的操作：``transformation和action``

::

        其实，如果大家有hadoop基础，为了理解方便的话，可以这样理解
        hadoop中的mr计算框架中包含map操作和reduce操作，
        spark计算框架中包含transformation操作和action操作
        但是注意：前期为了好理解可以暂且这样理解，其实这个解释是不对的，这个等后期熟悉了之后就可以区分开了。

1：transformation是得到一个新的RDD，方式很多，比如从数据源生成一个新的RDD，从RDD生成一个新的RDD

2：action是得到一个值，或者一个结果（直接将RDD cache到内存中）
所有的transformation都是采用的懒策略，就是如果只是将transformation提交是不会执行计算的，计算只有在action被提交的时候才被触发

RDD扩展操作
~~~~~~~~~~~~~~~

RDD ``actions和transfromations`` 可以执行更复杂的运算。假设我们单词最多的那行数据：

scala::

     scala> textFile.map(line => line.split(" ").size).reduce((a, b) => if (a > b) a else b)
     res4: Long = 15

首先是通过 ``map`` 函数把一行行数据映射成一个个数字类型的值，创建一个新的。 然后调用 ``reduce`` 函数获取到最大的那一行。 ``map`` 和 ``reduce`` 函数的参数是scala的闭包函数，并且还可以使用scala/java库中的功能。 
例如：在这里我们使用了``Math.max()`` 函数来获取最大值

::

    scala> import java.lang.Math
    import java.lang.Math

    scala> textFile.map(line => line.split(" ").size).reduce((a, b) => Math.max(a, b))
    res5: Int = 15

hadoop推出的一个常见的数据流模式是MapReduce，spark也可以很容易的实现MapReduce：

::

    scala> val wordCounts = textFile.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey((a, b) => a + b)
    wordCounts: org.apache.spark.rdd.RDD[(String, Int)] = ShuffledRDD[8] at reduceByKey at <console>:28

这里，我们结合 ``flatMap`` , ``map`` , ``reduceByKey`` 算子(函数)计算出了文件中的每个单词出现的次数，作为一个pair(String,Int)类型的RDD。``此处的pair可以理解为键值对类型的数据``
在我们的shell命令行下获取单词对应的次数数据，可以使用 ``collect`` 算子：

::

    scala> wordCounts.collect()
    res6: Array[(String, Int)] = Array((means,1), (under,2), (this,3), (Because,1), (Python,2), (agree,1), (cluster.,1), ...)

缓存(Caching)
~~~~~~~~~~~~~

spark还支持将一个数据集提交到集群的内存缓存中，这是非常有用的当这个数据被重复访问的时候，例如当查询一个小“热”数据集或运行像PageRank这种迭代算法。举一个简单的例子，让我们使用前面的 ``linesWithSpark`` 数据集来进行缓存：

::

    scala> linesWithSpark.cache()
    res7: linesWithSpark.type = MapPartitionsRDD[2] at filter at <console>:27

    scala> linesWithSpark.count()
    res8: Long = 15

    scala> linesWithSpark.count()
    res9: Long = 15

在这里，我们缓存了一个100行左右的文件，看起来好像没什么用，其实这些相同的函数可以用于非常大的数据集,即使他们跨越几十或几百个节点，
你可以通过 ``bin/spark-shell`` 这个工具来和spark集群交互，详细信息需要查看 `编程文档 <http://spark.apache.org/docs/latest/programming-guide.html#initializing-spark>`_ 。

spark应用开发
------------------

假设我们想使用sparkAPI来写一个应用，我们可以通过scala，java或者python来实现。

scala：
我们将会创建一个简单的spark应用代码，代码的文件名为：``SimpleApp.scala``

::

    /* SimpleApp.scala */
    import org.apache.spark.SparkContext
    import org.apache.spark.SparkContext._
    import org.apache.spark.SparkConf

    object SimpleApp {
      def main(args: Array[String]) {
        val logFile = "YOUR_SPARK_HOME/README.md" // 需要确保你的电脑中有这个文件，一定要修改YOUR_SPARK_HOME这个变量，改为你电脑上spark的安装目录
        val conf = new SparkConf().setAppName("Simple Application")
        val sc = new SparkContext(conf)
        val logData = sc.textFile(logFile, 2).cache()
        val numAs = logData.filter(line => line.contains("a")).count()
        val numBs = logData.filter(line => line.contains("b")).count()
        println(s"Lines with a: $numAs, Lines with b: $numBs")
        sc.stop()
      }
    }

请注意：这个应用的代码应该定义一个 ``main()`` 方法，而不是去继承 ``scala.App`` 。 ``scala.App`` 的子类可能无法正常运行。

这个程序仅仅统计了在spark目录里面 ``README.md`` 这个文件中有多少行包含字母 ``a`` 或者 包含字母 ``b`` 
请注意，你需要替换程序中的 ``YOUR_SPARK_HOME`` ,改为你的spark的安装目录，其实最终是为了确保能正确找到 ``README.md`` 这个文件
和之前在 ``spark-shell`` 下面写的代码不一样，在这里， ``SparkContext`` 对象是需要我们自己初始化的。

我们通过 ``SparkContext`` 的构造函数创建了一个 `SparkConf <http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkConf>`_  对象，这个对象里面包含了我们这个程序的一些基本信息。

我们的程序依赖sparkAPI，因此我们需要有一个sbt的配置文件 ``build.sbt`` ， 这个文件中需要添加spark的依赖。

::

    name := "Simple Project"

    version := "1.0"

    scalaVersion := "2.11.7"

    libraryDependencies += "org.apache.spark" %% "spark-core" % "2.1.1"
