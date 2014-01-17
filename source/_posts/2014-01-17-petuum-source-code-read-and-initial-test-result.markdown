---
layout: post
title: "Petuum: Source Code Read and Initial Test Result"
date: 2014-01-17 18:40:01 +0800
comments: true
categories: 
---

这几天为了测好[Petuum](http://petuum.org/)，花了一点时间看了一下Petuum源码，把其中的精华跟大家分享一下。

Petuum共有9050行代码，代码文件数39个。整个Petuum这么多源码，其实就只实现了一个LDA，外加一个Hello world。目前没有一个pull request和issue，另外已经很久（20天）没有更新了。发现C++写的在github上不是很受欢迎，GraphLab也很少有pull request。相比之下Spark的Pull request之多，热度完全不同。

**一级目录有：**

- Apps：LDA以及Hello world的具体实现

- Machinefiles：服务器worker的配置

- Scripts：启动/关闭job的脚本

- Src：主要的源码

- Third_party：编译时拉下来的第三方库

**Src下共有以下几个代码目录：**

- Comm_handler：主要是命令行参数解析，ZMQ配置等

- Consistency：一致性控制、一致性策略，以及操作日志记录

- Include：头文件集合

- Proxy：client代理和server代理。两者用来RPC通信，类似于我写的SSP中的akka框架中的一小部分功能

- Server：参数服务器，其实就是一些table的集合（允许多个参数服务器存在，可以进行参数的partition）

- Storage：table的存储，cache，以及还入换出策略

- Util：逻辑时钟vector clock，就是Dynamo中的策略，以及一些小组件

**以LDA为例（也没有别的例子），其大体逻辑如下：**

1. 初始化tablegroup，用于存储一系列table

2. 注册主线程

3. 在tableGroup中创建table

4. 创建LDA sampler

5. sampler读数据（直接读到内存中，而且是压缩格式，只读取字数总量）

6. 创建sampling的线程组

7. 在线程组创建线程，并绑定在runSampling的函数上

8. 执行这些线程

9. 关闭线程，table group，并结束

**整个过程中，8是实际干活的，也是唯一并行的地方。将这部分放大如下：**

1. 初始化每个线程拥有的数据，即数据分片，每个thread处理一片

2. 检查每个线程状态

3. 向参数服务器注册线程

4. 初始化topic

5. 进入sampling主循环

6. 结束，输出结果

**其中5是主要干活的，这里每个thread针对自己的一片文件进行sampling操作。该部分放大如下：**

1. 初始化wordsampler

2. 采样一次迭代

3. 计算似然度

4. barrier混合当前状态

**这里有一些取巧的地方，也是表现处SSP的地方。**

首先是初始化wordsampler的时候，需要从server获取最新的参数，这时参数请求不是发给server，而是发给本thread的cache，本thread cache合法则使用，否则使用本process的cache，合法则使用，否则才去server请求参数。（合法与否通过iteration的步子是否过于stale判断）。而向server请求参数也不是直接发送，是由clientProxy向serverProxy请求，serverProxy向server群体广播这个消息，拿到参数值。

其次是每次迭代之后首先更新本地cache，即read-my-write。之后混合当前状态，这个混合只是个“建议混合”，本质上是将本次操作的日志记录到opLog中。opLog中定义的向server更新的操作只有两个：INC和PUT，一个用于增量，一个用于修改。而每当table调用iterate函数的时候，会引导到consistency_controller类的DoIterate函数，随机触发背后clientProxy的sendOpLog函数，该函数通过RPC发送序列化之后的opLog给serverProxy，之后交由适当的server反序列化并apply到自身。
之后是一些注意事项：

- clientProxy 用作模拟RPC call，发送请求获得参数

- 只在headclient上进行doc likelihood计算，而每个线程计算自己的localwordlikelihood

- 真正对table的远程操作是在fast_word_sampler搞定的。

- clientproxy 负责SendOpLog 到server，而这个调用是在每次table调用Iterate的时候用到的。

- opLog的混合在server文件中

整个工程目前只有这些内容。其他的dynamic scheduler之类的统统没有。论文里号称matrix factorization，以及coordinate descent之类的测试也没见着。不过整体用C++写还是蛮挑战的，用scala+[akka](http://akka.io/)百来行就能实现差不多的功能。

之前一些简单的测试效果感觉不是很满意，例如stale增大后likelihood抖动很严重，而且效果对比BSP没有明显的变好太多。后续我会详细测试一下这个效果。
