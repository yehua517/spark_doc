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

上面代码中的 ``appName`` 是你应用的一个名字，可以在集群UI界面中展示。 ``master`` 是 `spark，mesos 或者 yarn 的集群地址 <http://spark.apache.org/docs/latest/submitting-applications.html#master-urls>`_，或者一个特殊的 ``local`` 字符串使用本地模式运行。在实际中，当在一个集群上运行，你将不会在程序中硬编码 ``master`` ，而是使用 `spark-submit <http://spark.apache.org/docs/latest/submitting-applications.html>`_ 脚本接收参数去运行程序。然而，针对本地测试代码和单元测试代码，你可以通过 ``local`` 在本地执行。

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


弹性分布式数据集 (RDDs)
--------------------

RDD是spark中非常重要的概念，它是一种可以并行操作的容错的元素集合。有两种方式去创建RDDs：1，在你的代码中初始化一个已经存在的集合 2，引用一个外部存储系统的数据集，例如一个共享的文件系统，hdfs ，hbase等...

并行化集合
~~~~~~~~~~

可以通过在你的程序代码中调用sparkContext的 ``parallelize`` 方法去操作一个已存在的集合来创建一个并行化集合。集合中的元素通过并行化操作复制到一个分布式的集合中可以被并行操作。例如：下面这个代码就是演示了通过数字1~5如何来创建一个并行化集合：

::

	val data = Array(1, 2, 3, 4, 5)
	val distData = sc.parallelize(data)

一旦创建成功，这个分布式的数据集就可以被并行化的操作。例如，我们可以调用 ``distData.reduce((a, b) => a + b)`` 这个方法来对数组中的数据进行聚合。我们一会将会描述对分布式数据集的操作。

并行集合的一个重要参数是对数据集的切片数量。spark将会在集群中针对每一个分片运行一个任务。通常，在你的集群中给每个cpu分配2-4个分片。正常情况下，spark会依据你的集群来自动设置分片的数量。然而，你也可以在 ``parallelize`` 这个方法的第二个参数中手工设置一个参数(例如： ``sc.parallelize(data, 10)`` )。注意：在代码中的一些地方会使用分片的同义词(term)来保证代码的向后兼容性。

外部数据集
~~~~~~~~~~

spark可以从任何hadoop支持的数据源来创建分布式数据集，包含你的本地文件，HDFS,Cassandra, HBase,  `Amazon S3 <http://wiki.apache.org/hadoop/AmazonS3>`_ , etc. spark支持文本文件(text files)，序列化文件( `SequenceFiles <http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html>`_ )和任意其他hadoop序列化( `inputFormat <http://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapred/InputFormat.html>`_ )的文件

文本文件的RDD对象可以通过 ``sparkContext`` 的 ``textFile`` 方法。这个方法需要文件的一个url(要么是一个本地文件，或者是hdfs://, s3n://, etc URI) 并且会把文件的所有行读取出来作为一个集合。下面是一个调用的例子：

::
	
	scala> val distFile = sc.textFile("data.txt")
	distFile: org.apache.spark.rdd.RDD[String] = data.txt MapPartitionsRDD[10] at textFile at <console>:26

一旦创建成功， ``distFile`` 就可以作为数据集来操作了。例如：我们可以统计所有行的大小，使用 ``map`` 和 ``reduce`` 方法： ``distFile.map(s => s.length).reduce((a, b) => a + b)``

spark读取文件的一些注意事项：

* 如果你使用一个本地文件系统路径，那么注意了，这个文件必须存在于集群的所有worker节点的相同路径下面。要么你把这个文件拷贝到所有的worker节点，或者使用一个共享的文件系统(例如 使用NFS实现文件共享)。

* spark的textFile方法，针对普通文本文件，支持一个目录，压缩文件，和通配符。例如：你可以使用 ``textFile("/my/directory")`` , ``textFile("/my/directory/*.txt")`` ,和 ``textFile("/my/directory/*.gz")`` 。

* 这个 ``textFile`` 方法需要一个可选的参数来控制文件的分片数量。默认情况下，spark针对文件的每一个block块创建一个分片(在hdfs中，block块的大小默认是128MB)，但是你也可以通过设置一个更大的值来获取更多的分片。注意：你设置的分片数量不能比blocks块的数量小。

除了文本文件类型，spark也支持几种其他数据格式：

*  ``SparkContext.wholeTextFiles`` 让你读取包含了很多小文件的目录，并且返回每一个小文件的信息(文件名，文件内容)作为一个pair。和文本类型文件对比，这个方法针对每一个文件只会返回一条记录(包含文件名和文件内容)。

* 针对 `序列化文件SequenceFiles <http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html>`_ ，使用sparkcontext的 ``sequenceFile[K, V]`` 方法，K和V是文件中key的类型和value的类型。这些应该是hadoop `Writable <http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/Writable.html>`_ 接口的子类，比如，`IntWritable <http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/IntWritable.html>`_ 和 `Text <http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/io/Text.html>`_ 。除此之外，spark也允许你使用一些本地类型针对常见的Writables；例如：``sequenceFile[Int, String]`` 会自动使用IntWritables 和 Texts。


* 针对其他hadoop格式数据格式，你可以使用 ``SparkContext.hadoopRDD`` 方法，它需要一个 ``JobConf`` 和输入格式化类，包含key的格式化类和value的格式化类。和设置hadoop任务类似的方式来设置你的数据源。你也可以使用 ``SparkContext.newAPIHadoopRDD`` 方法来使用新的 MapReduce API(org.apache.hadoop.mapreduce)来设置输入格式化类。

*  ``RDD.saveAsObjectFile``  和  ``SparkContext.objectFile`` 方法支持将普通序列化的java对象保存到RDD里面。虽然这种方式不像专业的序列化方式Avro那么高效，但是它提供了一个很简单的方式来保存任意RDD。


RDD操作
~~~~~~~~~~~

RDDs支持两种类型的操作： ``transformations`` (从一个已有数据集创建一个新的数据集) 和 ``actions`` (在一个数据集上进行一系列的运算之后把结果返回到driver程序)。例如， ``map`` 是一个transformation操作，它通过一个map函数对数据集中的每个元素进行操作，并且返回一个新的RDD作为结果。另一方面， ``reduce`` 是一个action操作，它通过一个reduce函数对RDD中的所有元素进行聚合操作，并且返回最终的聚合结果给driver程序。(还有一个 ``reduceByKey`` 方法可以返回一个分布式的数据集)。

 ``transformations`` 操作都是懒加载的，它们不会立刻进行计算。相反，它们只会记录一个哪些数据集需要执行这个transformation操作(例如，一个文件)。只有当我们执行action操作，必须要给driver返回一个结果的时候，这个transformation操作才会进行计算(因为单纯执行transformation操作没有意义，这个操作不会给driver返回结果，所以，只有调用了action操作才会真正执行transformation操作)。这种设计可以使spark更高效的执行。例如，

默认，当你对一个RDD进行一次action操作的时候，这个RDD都需要重新进行计算。然而，你也可以使用 ``persist`` (或者 ``cache`` )方法把一个RDD持久化到内存中，这样spark将会把这些数据保存到集群中，以便于当你下次查询的时候可以实现更快的访问。在这里也支持把RDD持久化到磁盘中，或者复制到集群的其他节点。

基本操作
^^^^^^^^^

为了演示RDD的基本操作，考虑使用下面的简单例子

::
	val lines = sc.textFile("data.txt")
	val lineLengths = lines.map(s => s.length)
	val totalLength = lineLengths.reduce((a, b) => a + b)

第一行代码表示读取一个外部文件来获得一个基本的RDD。这个数据没有加载到内存中或者其他地方， ``lines`` 仅仅是一个指向文件的指针。
第二行代码表示定义了一个 ``lineLengths`` 作为 ``map`` 操作的结果。由于懒加载的原因，``lineLengths`` 不是立即计算的。
最后，我们执行 ``reduce`` 函数，这个是一个action操作。这个时候，spark把这个计算任务分布到不同的机器上去运行，并且每台机器运行map的一部分和本地的聚合，最终把结果返回给driver程序。

如果我们以后还要使用 ``lineLengths`` ，我们可以执行下面代码

::
	lineLengths.persist()

在执行 ``reduce`` 函数之前，这将导致进行第一次计算之后把 ``lineLengths`` 保存到内存中。

函数操作
^^^^^^^^^

Spark’s API relies heavily on passing functions in the driver program to run on the cluster. There are two recommended ways to do this:

* Anonymous function syntax, which can be used for short pieces of code.
* Static methods in a global singleton object. For example, you can define object MyFunctions and then pass MyFunctions.func1, as follows:

::
	
	object MyFunctions {
	  def func1(s: String): String = { ... }
	}

	myRdd.map(MyFunctions.func1)

Note that while it is also possible to pass a reference to a method in a class instance (as opposed to a singleton object), this requires sending the object that contains that class along with the method. For example, consider:

::
	class MyClass {
	  def func1(s: String): String = { ... }
	  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(func1) }
	}

Here, if we create a new MyClass instance and call doStuff on it, the map inside there references the func1 method of that MyClass instance, so the whole object needs to be sent to the cluster. It is similar to writing rdd.map(x => this.func1(x)).

In a similar way, accessing fields of the outer object will reference the whole object:

::
	class MyClass {
	  val field = "Hello"
	  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(x => field + x) }
	}

is equivalent to writing rdd.map(x => this.field + x), which references all of this. To avoid this issue, the simplest way is to copy field into a local variable instead of accessing it externally:

::

	def doStuff(rdd: RDD[String]): RDD[String] = {
	  val field_ = this.field
	  rdd.map(x => field_ + x)
	}






