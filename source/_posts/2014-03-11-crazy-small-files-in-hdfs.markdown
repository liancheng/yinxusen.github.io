---
layout: post
title: "Crazy Small Files in HDFS"
date: 2014-03-11 16:28:47 +0800
comments: true
categories: 
---

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

**LDA best practice:**

I think there are two common ways to use LDA in practice. First is the use in experimental condition, say, you have a bunch of small files on your disk. Then you want to upload them into HDFS, and call LDA in Spark. This is a usual way if you just want to do some experiments with LDA. In other words, it is an off-line training process. The second way of using LDA is an industrial use. You may have a streaming pipe, which feeds new data from Twitter or some other websites into your system. You may choose to put those data into a distributed storage such as HDFS or HBase, or you just process the data stream.

Think them boldly, they are totally different. With respect to the usage of LDA, we should take care of the two scenarios simultaneously. Both of them are useful so we should not give up each of them.

**Offline scenario of LDA**

In the offline scenario, maybe you are not responsible for pre-processing. Instead, you just leave it to the end-user. Users transform the raw texts into the format you specify, and upload them into HDFS so your LDA application can read them directly. In this way, what we need to do is just specify the input format. What a relief !

Maybe you can help end-users one step more. You write a program, sequential or parallel, whichever is OK, to help the pre-processing for end-user. Just like what Mahout does. End-user may write an ugly shell program as coordinator, to control the overall workflow. In this way, you can write a program to transform the small files (raw texts) into a huge file which lines represents texts, with filenames in the front of the line plus a separator.

But, I think a better way is melding the pre-process with LDA. What the end-user does is just upload his raw texts on HDFS. In this way, we must provide the function to read all texts and their corresponding filenames in. Then we implement a `CombineFileInputFormat`, a `CombineFileRecordReader`, a `FileLineWritable` and an interface looks like `textFiles` to support the scenario.

I am not mean that it's a best practice. Indeed, it is very bad to put lots of small files on HDFS, for it will occupy so many index entries than bad performance will occur. I just talk about one feasible way. However, there are some tangle problems we must solve.

First of all is the block size of your HDFS. Although we mean "small files", but how small it is? Will its size larger than a single block in HDFS? The answer is Yes, it is possible. So we must handle the joint of blocks for each file, especially when the file is a multi-byte one, say, UTF encoded. Characters on the edge of blocks will be separated into two parts. We must take the responsibility to merge them together seamlessly.

The key point is, we do not like shuffle, especially the unnecessary one. We hope that blocks of each file could be stay in the same node, so that we can merge them together without shuffle. As with the [blog](http://www.idryman.org/blog/2013/09/22/process-small-files-on-hadoop-using-combinefileinputformat-1/) says, if we override the `isSplitable()` function and set the return value to false, then we can keep a single file in the same `split`. `HadoopRDD` treats a `split` as a single partition. If so, we just need to merge blocks of a single file in one partition without any shuffle. Very happy!

However, we find that what the blog says is wrong. The `isSplitable()` is really useful, but just for `FileInputFormat`, which is the parent class of `CombineFileInputFormat`. The latter has a very complicated logic to, you know, divide blocks into splits. However, in order to improve the read performance, `CombineFileInputFormat` constructs a node list and a rack list, to chain blocks in the same node, or the same rack together, and send them into the same split. Remember that HDFS has replications. If there are 3 replications of a block, then `CombineFileInputFormat` could have the possibility to read any of the 3 replications, so we cannot ensure that the class will read any blocks from any nodes! It seems that a global shuffle is inevitable.

Funny, we also find that there is a class named `MultiFileInputFormat`, which is the predecessor of `CombineFileInputFormat`. It is deprecated now, because of the low efficiency due to unawareness of data locality (per node / rack).

There must be some trade-offs here. We should solve the shuffle in HDFS level, or we need solve the shuffle in Spark level. If sizes of all files are smaller than the block size, there is nothing hard to solve. But we cannot assume things like that.

**Online scenario of LDA**

Now let's turn into another direction. Note that a online product will never take the way described above. You know, people comes from data process division would not be silly to save all raw texts in local disks, then upload it to servers when processing. Massive data, in my opinion, should be stay in an appropriate place. In this scenario, raw texts or web page should be stored in a KV store, such as HBase. Small texts or articles should be treated same with pictures from websites. So, in reality, HBase will be used. However, Spark has no external storage except HDFS and local disk. So I think it is the time to add new storage. 
 
**In case of using a our own customized partitioner**

Ah... It's awful! You know what, one of the reasons than Spark beats its counterparts is customized partitioners. It is the first time that you can arrange your items according to your wishes such easily. One idea to settle our problem is using a customized partitioner to rearrange our KV pairs. However, new partitioner is useful only when we do join of two RDDs iteratively, such as `PageRank`. But if we only need to shuffle things once time, it will be helplessness, for you cannot avoid the first-time shuffle.

**Where the shuffle from? How to do trade-off?**

We talked about needless shuffle just now. So question is, where is the shuffle from, and how to do trade-off? First, huge-files (or small files with smaller block size) could be cut off due to the fixed block size, so we want blocks belong to a single file could stay in the same split (partition, in dialect of Spark), then we can combine them together to recover the single file without any shuffle. The process is essential, because there will be multi-bytes characters such as UTF could be split. 

Second, `CombineFileInputFormat` cannot preserve the property for us, due to the consideration of efficiency (see `CombineFileInputFormat` and `MultiFileInputFormat` as an example). Mostly because of the replication in HDFS, the fault tolerance function in HDFS, blocks of a single file could be read at any nodes. 

So there are the trade-offs between "shuffle HDFS level" with "shuffle Spark level", and between "efficiency when reading blocks" with "efficiency due to shuffle-free", and eventually between "efficiency" with "security".

**Rethink the design of file system, efficiency and robustness**

**Rethink the compact code and black magic behind it**

