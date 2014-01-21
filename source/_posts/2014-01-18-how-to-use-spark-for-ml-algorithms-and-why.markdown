---
layout: post
title: "How to use Spark for ML algorithms and why ?"
date: 2014-01-18 16:33:43 +0800
comments: true
categories: 
---
**NOTE** This PR is only a request for comments, since it introduces some minor incompatible interface change in MLlib.

**Update 2014-01-16** The inner iteration counts of local optimization is also an important parameter, which is related to the convergence rate. I will add some new experiments about it ASAP. 

**Update 2014-01-16 [2]**Using `data.cache` brings a great performance gain, BSP+ is worse than original version then.

**Update 2014-01-17** When we removing the straggler of BSP+, BSP+ is better than original version. Straggler comes from the `sc.textFile`, HDFS gives bad answer. Seems that SSP is more reasonable and useful now. Besides, inner iteration is also a big factor. For our data with 15 partitions, 60 seems to be the best inner iteration. 

If there is no straggler at all, the costs caused by framework must be higher than the inner iteration expansion. Meanwhile, the uncertainty caused by high parallelism is made up by the acceleration.

**Update 2014-01-18** We also find that there are some influences come from the partition number. As we said earlier, there is a inflection point.

**Update 2014-01-18 [2]** We test SVM with BSP+, it runs cool. We also modify LASSO, RidgeRegression, LinearRegression.

**Update 2014-01-18 [3]** BSP+ SVM beats original SVM 7 倍，是不是JobLogger或者时间的统计会影响性能？因为后者打印的log数量非常庞大。经过验证，阎栋加入的JobLogger没有那么严重的影响。由系统加入的TaskLog和DAGLog不知道怎么停。

**Update 2014-01-18 [4]** 思考一个问题，为什么同样的工作量，60次混合会比传统的梯度下降要好？要能解释这一点。差异只在混合策略上，例如，我有一个想法，还没想清楚呢就跟别人说了，搞得大家都不明白。如果自己想清楚了，再跟别人说会更明白。

**Update 2014-01-18 [5]** BSP+快的原因，因为同步次数少了，导致网络开销同比减少。所以结果比原始情况好。大大的提升通信量，才能展现出我们的优势。

**Update 2014-01-18 [6]** 找了一个新数据，这份数据2000维度，30多个GB，比之前的unigram好，但又比trigram少，可见mllib之废物，1000w的维度就已经跪了！！这还做毛个大数据啊？本来像自己动手生成数据集，但是总感觉不好。网上找到一个新的。找新数据的目的就是增加维度，这样让每次迭代之间传输的数据量更大，我们的优势更加明显。

**factors we found**

- number of partitions
- straggler (YJP profiling)
- inner iteration
- outer iteration

**Two different usages of Spark present two different thoughts**

- The classic one is that we use Spark as a distributed code compiler, plus with a task dispatcher and executors. In this way, [Kay Ousterhout](http://www.eecs.berkeley.edu/~keo/) publish a paper called [Sparrow: Distributed, Low Latency Scheduling](http://www.cs.berkeley.edu/~matei/papers/2013/sosp_sparrow.pdf) is the future. However, I don't think it is the best practice of Spark. The [DAG scheduler stack overflow](https://spark-project.atlassian.net/browse/SPARK-1006) is also a big question as mentioned by [Matei Zaharia](http://www.cs.berkeley.edu/~matei/).

- A more natural way to use Spark W.R.T. machine learning is treat Spark as a effective distributed executive container. Data with cache stay in each executor, computing flow over these data, and feedback parameters to drivers again and again.

## Introduction

In this PR, we propose a new implementation of `GradientDescent`, which follows a parallelism model we call BSP+, inspired by Jeff Dean's [DistBelief](http://research.google.com/archive/large_deep_networks_nips2012.html) and Eric Xing's [SSP](http://petuum.org/research.html).  With a few modifications of `runMiniBatchSGD`, the BSP+ version can outperform the original sequential version by about 4x without sacrificing accuracy, and can be easily adopted by most classification and regression algorithms in MLlib.

Parallelism of many ML algorithms are limited by the sequential updating process of optimization algorithms they use.  However, by carefully breaking the sequential chain, the updating process can be parallelized.  In the BSP+ version of `runMiniBatchSGD`, we split the iteration loop into multiple supersteps.  Within each superstep, an inner loop that runs a local optimization process is introduced into each partition.  During the local optimization, only local data points in the partition are involved.  Since different partitions are processed in parallel, the local optimization process is natually parallelized.  Then, at the end of each superstep, all the gradients and loss histories computed from each partition are collected and merged in a bulk synchronous manner.

This modification is very localized, and hardly affects the topology of RDD DAGs of ML algorithms built above.  Take `LogisticRegressionWithSGD` as an example, here is the RDD DAG of a 3-iteration job with the original sequential `GradientDescent` implementation:

![123](https://f.cloud.github.com/assets/2637239/1901663/dbd44be0-7c67-11e3-8c44-800a10f6d92a.jpg "Original version of `LogisticRegressionWithSGD`")

**Figure 1. RDD DAG of the original LR (3-iteration)**

And this is the RDD DAG of the one with BSP+ `GradientDescent`:

![234](https://f.cloud.github.com/assets/2637239/1901664/e5fea980-7c67-11e3-9e24-5c9978d94d02.jpg "BSP+ version of `LogisticRegressionWithSGD`")

**Figure 2. RDD DAG of the BSP+ LR (3-iteration)**

## Experiments

To profile the accuracy and efficiency, we have run several experiments with both versions of `LogisticRegressionWithSGD`:

*   Dataset: the unigram subset of the public [web spam detection dataset](http://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/binary.html)

    *   Sample count: 350,000
    *   Feature count: 254
    *   File size: 382MB

*   Hardware:

    *   Nodes count: 15
    *   CPU core count: 120

*   Spark memory configuration:

    *   `SPARK_MEM`: 8g
    *   `SPARK_WORK_MEMORY`: 10g

Experiment results are presented below.

### Rate of convergence

![08](https://f.cloud.github.com/assets/2637239/1909932/affb2118-7d09-11e3-8b59-abe2584d88cd.png)

**Figure 3. Rate of convergence**

![graph3](https://f.cloud.github.com/assets/2637239/1917187/9a808e16-7d8d-11e3-8e8e-0d279d7f5cbc.png)

**Figure 3.1 Rate of convergence W.R.T. time elapsed**

Experiment parameters:

*   BSP+ version:

    *   Superstep count: 20
    *   Local optimization iteration count: 20

*   Original version:

    *   Iteration count: 20

Notice that in the case of BSP+, actually `20 * 20 = 400` iterations are computed, but the per-partition local optimization iterations are executed *in parallel*.  From figure 3 we can see that the BSP+ version converges at superstep 4 (80 iterations), and the result after superstep 3 is already better than the final result of the original LR. From figure 3.1 we can get a more clear insight of the speedup.

![07](https://f.cloud.github.com/assets/2637239/1909937/d5dc6248-7d09-11e3-922f-89fcc4431ef0.png)

**Figure 4. Iteration/superstep time**

Next, let's see the time consumption.  Figure 4 shows that single superstep time of BSP+ LR is about 1.6 to 1.9 times of single iteration time of the original LR.  Since the final result of original LR doesn't catch up with superstep 3 of BSP+ LR, we may conclude that BSP+ is at least `(20 * 6 * 10^9 ns) / (3 * 1.2 * 10^10 ns) = 3.33` times faster than the original LR. Actually it has 4.3x performance gain in comparison with original LR, as depicted in figure 3.1. The main reason is that: the original version submits 1 job per iteration, while the BSP+ version submits 1 job per superstep, and per partition local optimization doesn't involve any job submission.

### Correctness

![09](https://f.cloud.github.com/assets/2637239/1909941/f05122b2-7d09-11e3-84b4-10a81ac0b14a.png)

**Figure 5. Loss history**

Experiment parameters:

*   BSP+ version:

    *   Superstep count: 20
    *   Local optimization iteration count: 20

*   Original version:

    *   Iteration count: 80

In this experiment, we compare the loss histories of both versions of LR.  We can see that BSP+ gives better answer much faster.

### Relationship between parallelism and the rate of convergence

![10](https://f.cloud.github.com/assets/2637239/1909944/1379794c-7d0a-11e3-8a1f-7e3401422cf7.png)

**Figure 6. Iteration/superstep time under different #partitions**

![13](https://f.cloud.github.com/assets/2637239/1909945/2044fa70-7d0a-11e3-811d-359c20e2e0d6.png)

**Figure 7. Job time under different #partitions**

Experiment parameter:

*   BSP+ version:

    *   Superstep count: 20
    *   Local optimization iteration count: 20

*   Original version:

    *   Iteration count: 20

In the case of BSP+, by adjusting minimal number of partitions (actual partition number is decided by the `HadoopRDD` class), we can explore the relationship between parallelism and the rate of convergence.  From figure 6 and figure 7 we can see, not surprisingly, single iteration/superstep time and job time decrease when number of partitions increases.

![14](https://f.cloud.github.com/assets/2637239/1909947/34517656-7d0a-11e3-90bd-029cf802e35a.png)

**Figure 8. Job time under different #partitions.  Each job converges to roughly the same level.**

Experiment parameter:

*   BSP+ version:

    *   Local optimization iteration count: 20
    *   All jobs runs until they converges to roughtly the same level

Then follows the interesting part.  In figure 8, several jobs are executed under different number of partitions.  By adjusting superstep count, we make all jobs converges to roughly the same level, and compare their job time.  The figure shows that the job time is a convex curve, whose inflection point occurs when #partition is 45.  So here is a trade off between parallelism and the rate of convergence: we cannot always increase the rate of convergence by increasing parallelism, since more partition implies fewer sample points within a single partition, and poorer accuracy for the parallel local optimization processes.

## Acknowledgement

Thanks @liancheng for the prototype implementation of the BSP+ SGD.
