# 概述

由于深度学习的训练任务动辄几小时、数天，甚至上周、好几个月，所以分布式和GPU是深度学习不可缺少的。

本文主要介绍多机多卡分布式的书写逻辑，以及在DLS上运行的注意事项。

本模块将会以经典的例子：MNIST手写数字识别 来带你快速入门TensorFlow的分布式写法，并了解如何在DLS上运行，包括：

- TensorFlow分布式的分类
- TensorFlow多机多卡实现思路
- 在DLS运行的注意事项

本文档中涉及的演示代码和数据集来源于网络，你可以在这里下载到：[DIST_MNIST.zip](https://s3.meituan.net/v1/mss_0e5a0056f3b64f79aa4749ffa68ce372/cms001/DIST_MNIST-20171220.zip)

# TensorFlow分布式的分类
深度学习的分布式总共可以分为两大类：

* 图间并行
* 图内并行

##图间并行（又称数据并行）：
每个机器上都会有一个完整的模型，将数据分散到各个机器，分别计算梯度。
##图内并行（又称模型并行）：
每个机器分别负责整个模型的一部分计算任务。

本文采用的分布式是图间并行，图间并行又根据梯度更新方式分为两类：

* 同步：收集到足够数量的梯度，一同更新
* 异步：即同步方式下，更新需求的梯度数量为1

本文两种方式都实现了，可以通过设置启动参数来设置使用哪儿种并行方式。
值得提一下的是，本文的实现采用的是PS这种通信方式。

PS(parameter server):维护全局共享的模型参数的服务器。

![alt text](https://s3.meituan.net/v1/mss_0e5a0056f3b64f79aa4749ffa68ce372/cms001/sync_async_tensorflow_diagram-1513567409120.png =400x400 "图间并行的同步和异步方式示意图")


# TensorFlow多机多卡实现思路
多级多卡的分布式有很多实现方式，比如：

1. 将每个GPU当做一个worker
2. 同一个机器的各个GPU进行图内并行
3. 同一个机器的各个GPU进行图间并行
4. 其他

本文采用的是第3种方式。
其具体方式为：

1. 将模型实现封装成函数
2. 将数据分成GPU数量的份数
3. 在每个GPU下，进行一次模型forward计算，并使用优化器算出梯度
4. reduce每个GPU下的梯度，并将梯度传入到分布式中的优化器中

分布式的优化器是指如果是同步方式，是需要将优化器传入到tf.train.SyncReplicasOptimizer中。异步则不需要。


# 在DLS运行的注意事项
**注：请选择TensorFlow1.10版本的镜像。**
##PS不使用server.join
一般TensorFlow的程序，包括TensorFlow的官方文档都对PS采用了server.join的处理方式，但是这种方式处理有一个问题：PS无法随着训练的结束而自动停止，需要手动kill，很麻烦。
如果想在DLS运行，则无法做到手动kill PS的，因此需要对PS进行一种特殊实现，保证在各个worker退出后，PS会自动退出，结束训练任务。

具体的做法：

采用队列通信的方式同步。

1. 为PS和每个worker创建一个队列
2. PS在server创建后，对队列进行worker数量的dequeue操作
3. 每个worker在完成后，对队列进行一次enqueue操作

##不需要传入和分布式有关的启动参数
由于DLS系统会自动分配资源和调度，同时会将相应的信息在程序启动时通过命令行参数传入，因此在创建分布式的TensorFlow任务的时候，不需要在启动参数一栏填入以下信息：

* task_index
* ps_hosts
* worker_hosts
* job_name

但是就本示例的程序而言，num_gpus参数需要手动传入命令行，这个参数表明每个worker可以使用多少GPU。

![alt text](https://s3.meituan.net/v1/mss_0e5a0056f3b64f79aa4749ffa68ce372/cms001/input_output_config-1513567407103.png =655x379 "输入输出配置")

![alt text](https://s3.meituan.net/v1/mss_0e5a0056f3b64f79aa4749ffa68ce372/cms001/compute_resource-1513567406961.png =622x291 "计算资源")


**当然示例代码中还实现了其他的一些功能，这里就不做详细的描述了。可以直接阅读代码，如果发现代码缺陷或者有不明白之处欢迎交流。**

**祝您TensorFlow之旅愉快，祝好！**
