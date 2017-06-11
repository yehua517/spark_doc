spark 编程指南
=============

* 概述
* 使用saprk
* 初始化Spark
	* 使用spark-Shell
* 弹性分布式数据集 (RDDs)
	* 并行化集合
	* 外部数据集
	* RDD操作
		* 基本操作
		* 函数操作
		* 理解闭包
			* 例子
			* 本地模式 VS 集群模式
			* 输出RDD的元素
		* 使用key-value键值对(tuple)
		* Transformations
		* Actions
		* Shuffle操作
			* 背景介绍
			* 对性能的影响
	* RDD持久化
		* 选择哪种存储策略?
		* 移除掉存储的数据
* 共享变量
	* 广播变量(Broadcast)
	* 累加器(Accumulators)
* spark集群部署
* 使用Java或者Scala启动spark程序
* spark单元测试
* 更多内容链接

概述
---------

通常情况下，每一个spark程序都是由一个驱动程序去运行用户的 ``main`` 函数并且在集群上执行各种并行操作。一个集合的元素可以跨越集群的节点并行处理，最主要是因为spark提供了一个弹性分布式数据集，简称为 ``RDD`` 。RDDs可以使用hadoop中的文件创建(或者其他类似的文件系统)，或者使用driver程序中存在的scala集合，然后对它进行操作。用户也可以要求spark在内存中持久化一个RDD，允许它在并行操作中有效的重用。最后，RDDs可以自动从故障的节点中恢复。

spark的第二个抽象是共享变量也可以进行并行操作。默认，当spark在不同的节点的任务中并行运行一个集合的时候，它会复制这个集合到每一个任务中。有时候，一个变量需要跨任务共享，或者在任务和驱动之间共享。
spark提供两种类型的共享变量::

	广播变量(Broadcast)：可以在所有的节点的内存中缓存一个值
	累加器(Accumulators)：这个变量只有增加，例如求总数和求和

它是很容易的如果你使用spark提供的交互shell工具,如果你是使用scala，那么使用 ``bin/spark-shell`` 。如果你是使用python，则需要使用 ``bin/pyspark`` 。但是本文档不翻译python的操作。

使用saprk
-------------

spark 2.1.1 默认是在scala2.1.1下构建和工作的。(spark也可以使用其他版本的scala)，如果想使用scala写spark程序，你需要使用兼容的scala版本(例如：2.11.x系列版本)

写一个spark程序，你需要添加一个spark的maven依赖。可以去maven仓库中查找spark依赖：

::
	
	groupId = org.apache.spark
	artifactId = spark-core_2.11
	version = 2.1.1

除此之外，如果你希望访问一个HDFS集群，你还需要添加一个和你的hdfs相同版本的hadoop-client依赖。

::

	groupId = org.apache.hadoop
	artifactId = hadoop-client
	version = <your-hdfs-version>

Finally, you need to import some Spark classes into your program. Add the following lines:
最后，你需要在你的程序中导入一些spark的包，添加下面这几行：

::

	import org.apache.spark.SparkContext
	import org.apache.spark.SparkConf

(在spark 1.3.0 以前，你需要显式的导入import org.apache.spark.SparkContext._ 来开启隐式转换功能)

初始化Spark
-------------

一个spark程序首先要做的第一件事情是需要创建一个 `sparkContext <http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkContext>`_ 对象，这个告诉了集群如何连接到集群。在创建sparkContext对象之前需要先创建一个包含应用基本信息的 `sparkConf <http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkConf>`_ 对象。

一个JVM中只能有一个活跃的sparkContext。如果你想创建一个新的sparkContext对象的话必须先调用 ``stop()`` 把之前的sparkContext停止掉

::

	val conf = new SparkConf().setAppName(appName).setMaster(master)
	new SparkContext(conf)

上面代码中的 ``appName`` 是你应用的一个名字，可以在集群UI界面中展示。 ``master`` 是 `spark，mesos 或者 yarn 的集群地址 <http://spark.apache.org/docs/latest/submitting-applications.html#master-urls>`_，或者一个特殊的 ``local`` 字符串使用本地模式运行。在实际中，当在一个集群上运行，你将不会再程序中硬编码 ``master`` ，而是使用 `spark-submit <http://spark.apache.org/docs/latest/submitting-applications.html>`_ 脚本接收参数去运行程序。然而，针对本地测试代码和单元测试代码，你可以通过 ``local`` 在spark进程内部执行。

使用spark-Shell
~~~~~~~~~~~~~~~~

在spark shell里面，一个特殊的SparkContext已经给你准备好了，在变量里面称为 ``sc`` 。使用你自己创建的SparkContext将不能正常使用。你可以使用 ``--master`` 参数来指定连接到哪个master，并且你还可以通过给 ``--jars`` 参数设置一个以逗号分隔的文件列表来把一些jar包添加到类路径(classpath)下面。你也可以添加依赖到你的shell会话中通过给 ``--packages`` 参数设置一个逗号分割的列表。任何额外的依赖可能存在的仓库可以通过 ``--repositories`` 参数指定。例如，运行 ``bin/spark-shell`` 在四个cpu core上，使用：

::

	$ ./bin/spark-shell --master local[4]

或者，也可以添加 ``code.jar`` 到类路径(classpath),使用：

::

	$ ./bin/spark-shell --master local[4] --jars code.jar

使用maven的方式包含一个依赖：

::
	
	$ ./bin/spark-shell --master local[4] --packages "org.example:example:0.1"

如果想要查看一个完整的列表，运行 ``spark-shell --help`` 。在这个场景下， ``spark-shell`` 调用了更多普通的 `spark-submit函数 <http://spark.apache.org/docs/latest/submitting-applications.html>`_
