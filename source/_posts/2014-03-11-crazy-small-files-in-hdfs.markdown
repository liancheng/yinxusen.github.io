---
layout: post
title: "Crazy Small Files in HDFS"
date: 2014-03-11 16:28:47 +0800
comments: true
categories: 
---

## Background

2 months ago, I intent to contribute a LDA algorithm to Spark, coordinate with my parallel machine learning paper. After I finished the core of LDA - the Gibbs sampling, I find that there are some trivial matters in the way of creating a usable LDA. Mostly, they are the pre-processing of text files. For the word segmentation, both Chinese and English, I wrap Lucene with a piece of scala code to support that. But the input format traps me lots of time.

The standard input format of Spark is from the interface called `textFiles(path, miniSplit)` in the `SparkContext` class. But it is a line processor, which digest one line each time. But what I want is a KV processor, i.e. I need an interface which can return me a KV pair (fileName, content) given a directory path. So I try to write my own `InputFormat`.

Firstly, I try to use the `lineReader` and handle the fragments of blocks myself, later I find that it's both ugly and unnecessary. So I use a more low level interface named `FSDataInputStream` to read an entire block once time. However, there are still some details need to be improved. Here, let's begin our explore. 

<!--more-->

## LDA best practice:

I think there are two common ways to use LDA in practice. First is the use in experimental condition, say, you have a bunch of small files on your disk. Then you want to upload them into HDFS, and call LDA in Spark. This is a usual way if you just want to do some experiments with LDA. In other words, it is an off-line training process. The second way of using LDA is an industrial use. You may have a streaming pipe, which will transport new feeds in Twitter or some other websites into your system. You will choose to put those feeds into a distributed storage such as HDFS or HBase, or you just send the streaming into a process.

Think them boldly, they are totally different. With respect to the usage of LDA, we should take care of the two scenarios simultaneously. Both of them are useful so we should not give up each of them.

## Offline scenario of LDA

In the offline scenario, you may not charge of the pre-processing, instead you just leave it to the end-user. Users will change the raw texts into the format you want, and upload them into HDFS so your LDA application can read them directly. In this way, what we need to do is just specify the input format. What a relief !

Maybe you can help end-users one step more. You write a program, single or parallel, whatever, to help the pre-process for end-user. Just like what Mahout does. End-user should write a ugly shell program as coordinator, to control the overall workflow. In this way, you can write a program to change the small files (raw texts) into a huge file which lines represents texts, with filenames in the front of the line plus a separator.

But, I think a better way is melding the pre-process with LDA. What the end-user does is just upload his raw texts on HDFS. In this way, we must provide the function to read all texts and their corresponding filenames in. Then we implement a `CombineFileInputFormat`, a `CombineFileRecordReader`, a `FileLineWritable` and an interface looks like `textFiles` to support the scenario.

I am not mean that it's a best practice. Indeed, it is very bad to put lots of small files on HDFS, for it will occupy so many index entries than bad performance will occur. I just talk about one feasible way. However, there are some tangle problems we must solve.

First of all is the block size of your HDFS. Although we mean "small files", but how small it is? Will its size larger than a single block in HDFS? The answer is Yes, it is possible. So we must handle the joint of blocks for each file, especially when the file is a multi-byte one, say, UTF encoded. Characters on the edge of blocks will be separated into two parts. We must take the responsibility to merge them together seamlessly.

The key point is, we do not like shuffle, especially the unnecessary one. We hope that blocks of each file could be stay in the same node, so that we can merge them together without shuffle. As with the [blog](http://www.idryman.org/blog/2013/09/22/process-small-files-on-hadoop-using-combinefileinputformat-1/) says, if we override the `isSplitable()` function and set the return value to false, then we can keep a single file in the same `split`. `HadoopRDD` treats a `split` as a single partition. If so, we just need to merge blocks of a single file in one partition without any shuffle. Very happy!

However, we find that what the blog says is wrong. The `isSplitable()` is really useful, but just for `FileInputFormat`, which is the parent class of `CombineFileInputFormat`. The latter has a very complicated logic to, you know, divide blocks into splits. However, in order to improve the read performance, `CombineFileInputFormat` constructs a node list and a rack list, to chain blocks in the same node, or the same rack together, and send them into the same split. Remember that HDFS has replications. If there are 3 replications of a block, then `CombineFileInputFormat` could have the possibility to read any of the 3 replications, so we cannot ensure that the class will read any blocks from any nodes! It seems that a global shuffle is inevitable.

Funny, we also find that there is a class named `MultiFileInputFormat`, which is the predecessor of `CombineFileInputFormat`. It is deprecated now, because of the low efficiency due to unawareness of data locality (per node / rack).

There must be some trade-offs here. We should solve the shuffle in HDFS level, or we need solve the shuffle in Spark level. If sizes of all files are smaller than the block size, there is nothing hard to solve. But we cannot assume things like that.

## Online scenario of LDA

Now let's turn into another direction. Note that a online product will never take the way described above. You know, people comes from data process division would not be silly to save all raw texts in local disks, then upload it to servers when processing. Massive data, in my opinion, should be stay in an appropriate place. In this scenario, raw texts or web page should be stored in a KV store, such as HBase. Small texts or articles should be treated same with pictures from websites. So, in reality, HBase will be used. However, Spark has no external storage except HDFS and local disk. So I think it is the time to add new storage. 
 
## In case of using a our own customized partitioner

Ah... It's awful! You know what, one of the reasons than Spark beats its counterparts is customized partitioners. It is the first time that you can arrange your items according to your wishes such easily. One idea to settle our problem is using a customized partitioner to rearrange our KV pairs. However, new partitioner is useful only when we do join of two RDDs iteratively, such as `PageRank`. But if we only need to shuffle things once time, it will be helplessness, for you cannot avoid the first-time shuffle.

## Where the shuffle from? How to do trade-off?

We talked about needless shuffle just now. So question is, where is the shuffle from, and how to do trade-off? First, huge-files (or small files with smaller block size) could be cut off due to the fixed block size, so we want blocks belong to a single file could stay in the same split (partition, in dialect of Spark), then we can combine them together to recover the single file without any shuffle. The process is essential, because there will be multi-bytes characters such as UTF could be split. 

Second, `CombineFileInputFormat` cannot preserve the property for us, due to the consideration of efficiency (see `CombineFileInputFormat` and `MultiFileInputFormat` as an example). Mostly because of the replication in HDFS, the fault tolerance function in HDFS, blocks of a single file could be read at any nodes. 

So there are the trade-offs between "shuffle HDFS level" with "shuffle Spark level", and between "efficiency when reading blocks" with "efficiency due to shuffle-free", and eventually between "efficiency" with "security".

## Take a deep breath - Full disclosure of locaility in Hadoop

To get into the secret of locaility of Hadoop IO, I have to look deep into `InputFormat` code in `mapred`. Due to the use of Spark, I choose `FileInputFormat` as the breach. First you should keep these concepts in mind, which will be used commonly later. They are *rack*, *node*, *file*, *block*, *replica*. A rack is composed of several nodes, nodes are machines composing HDFS in Hadoop. A file is composed of several blocks. A block could have several replicas, usually 3 copies. Note that your Hadoop workers could cover all HDFS nodes, but there could also mismatch between Hadoop workers and HDFS nodes. Note also that replicas of a block are usually span different racks, due to the consideration of robustness.

Things could be a little bit more complicated, if we add the workers of Hadoop in. Program could span across different workers, data could span across different nodes. So, question is, how to arrange the mapping of programs in each worker and blocks in each node, to get the best locaility, i.e. the less network communication when reading files on HDFS?

This is not easy, since there are many layers between program with block. Program is aware of file directly. File divided into several blocks. Block could be located in each nodes, and its replicas could be located in any other nodes. Different nodes could in different racks. Let's begin from our program. Take Spark as an example, you may call `hadoopRDD = sc.textFile(path)` to tell Spark read a file in. The path could be a local disk path, or more commonly, a HDFS path. `hadoopRDD` is usually partitioned for distributed computing. So, where is the partition information from? The answer is `Split` in HDFS. `Split`, or more specifically, `FileSplit`, which is used in `FileInputFormat`. `FileSplit` is an approach to arrange the mapping of **blocks and programs**.

Each `FileSplit` is a block set, in which blocks will be computed in the same worker, i.e. they are partitioned together. To preserve the locaility, `FileSplit` takes lots of efforts to place appropriate blocks together. Such as contribution computing, and node <-> block, rack <-> block double linked lists, etc. Note that shuffle in Spark is only related to the `Split`, because `Split` serves as a layer to shield the details in HDFS, which means that, if and only if we put blocks of an entire file into the same `Split`, our Spark is then "shuffle-free". But we cannot arrange different small files into a split in an random order, because different small files could be in everywhere on the HDFS cluster. If we put two files which are far apart in the same `Split`, bad performance will occur. Here we degenerate the program - block mapping to **split - block** mapping.

### contribution computing

We should remind us that there is little chance to put blocks in the same node in the same `Split`, because we cannot directly access blocks, instead we just specify the file path. Suppose that we have a `Split`, in which there are 3 blocks, and come from 8 nodes. 8 nodes belong to 4 racks. Moreover, each block has 3 replicas in total. Let's assume that the lengths of 3 blocks are 100, 150, 75, respectively. How to arrange the `perferedLocation` in this scenario? Namely, on which workers should the `Split` be processed?

![pic-1](/images/2014/03/pic-1.png)

First of all, we all agree that the `preferedLocation` should be a subset of all nodes of our block. In our example, it would be a subset of [h1 ... h8]. The second is, how to sort the subset, so as to make "the best" node at first, then "the second-best" one, ... 

There are two different ways to arrange the `preferedLocation`- the rack-aware way and the rack-free way.

### Double linked lists

**New Design and Implementation**



key ideas:

- 癫狂小文件：由small files input探索HDFS：功用、功效、策略

- Spark的处理方式：partition自己定制，把O(n^2)的shuffle变成O(n)的？

- Hbase作为替代

- Hdfs combine file与multi file对比，后者为什么倍deprecated

- Shuffle为什么存在，如何避免，是在底层（HDFS）还是在上层（Spark应用）？

- 偶尔要为replication容错付出更多代价

- Combine file input format，blog上错误多影响广，以及不友好的API，糟糕的示例和注释。

- 实际场景？学术研究VS线上应用。什么才是LDA输入的最佳实践？Mahout的处理方式？

- 文件系统的设计思考，可引入我自己的文件系统。

- 代码的简洁如何保证？同时如何保证性能？隐藏在简洁代码中的黑魔法。

- Partition，split，以及种种locaility preserve的思考（为什么程序性能这么差？1024个partition从何而来？）

- Google在bigtable和GFS的思考

- LineRecorder中的readline函数如何实现越过block向后看？
