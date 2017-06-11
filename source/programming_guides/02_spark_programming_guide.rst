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
	* 广播变量
	* 累加器
* spark集群部署
* 使用Java或者Scala启动spark程序
* spark单元测试
* 更多内容链接

概述
---------

通常情况下，每一个spark程序都是由一个驱动程序去运行用户的 ``main`` 函数并且在集群上执行各种并行操作。一个集合的元素可以跨越集群的节点并行处理，最主要是因为spark提供了一个弹性分布式数据集，简称为 ``RDD`` 。RDDs可以使用hadoop中的文件创建(或者其他类似的文件系统)，或者使用driver程序中存在的scala集合，然后对它进行操作。用户也可以要求spark在内存中持久化一个RDD，允许它在并行操作中有效的重用。最后，RDDs可以自动从故障的节点中恢复。

spark的第二个抽象是共享变量也可以进行并行操作。默认，当spark在不同的节点的任务中并行运行一个集合的时候，它会复制这个集合到每一个任务中。有时候，一个变量需要跨任务共享，或者在任务和驱动之间共享。
spark提供两种类型的共享变量::

	广播变量：可以在所有的节点的内存中缓存一个值
	累加器：这个变量只有增加，例如求总数和求和

它是很容易的如果你使用spark提供的交互shell工具,如果你是使用scala，那么使用 ``bin/spark-shell`` 。如果你是使用python，则需要使用 ``bin/pyspark`` 。但是本文档不翻译python的操作。

使用saprk
-------------

Spark 2.1.1 is built and distributed to work with Scala 2.11 by default. (Spark can be built to work with other versions of Scala, too.) To write applications in Scala, you will need to use a compatible Scala version (e.g. 2.11.X).
