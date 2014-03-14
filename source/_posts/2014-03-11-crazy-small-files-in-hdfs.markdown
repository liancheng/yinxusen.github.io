---
layout: post
title: "Crazy Small Files in HDFS"
date: 2014-03-11 16:28:47 +0800
comments: true
categories: 
---

## Background

2 months ago, I intended to contribute an LDA implementation to Spark, as part of my work of my parallel machine learning paper. After I finished the core part of LDA -- the Gibbs sampling, I found out that there were some subtle issues in the way of implementing a practical version of LDA. Mostly in input text files pre-processing. I wrapped Lucene with some Scala code to support Chinese and English word segmentation, just like what [ScalaNLP](http://www.scalanlp.org/) does. But the Hadoop input format issue bothered me a lot.

The standard input format used by Spark is `TextLineFormat`, which appears in `SparkContext.textFiles(path: String, miniSplit: Int)`. But it is a line-based, which returns one text line per record. However, what I want is a KV processor, i.e., I need an interface which returns a bunch of KV pairs in the format of `(fileName, content)` given path to a directory containing all the input text files. So I tried to write my own `InputFormat`.

Firstly, I tried to use `LineRecordReader` and handle fragments of blocks by myself. Later I found out that it was both ugly and unnecessary, just as the code listed below. I have to glue them together with a fixed separator - `'\n'`. Instead of that, I use a more low level interface named `FSDataInputStream` to read an entire block once time. However, there are still some details need to be improved. Here, let's begin our explore. 

{% codeblock lineReader version RecordReader (the terrible version) - BatchFileRecordReader.java lang:java%}

    /**
     * Reads an entire block contents. Note that files which are larger than the block size of HDFS
     * are cut by HDFS, then there are some fragments. File names and offsets are keep in the key,
     * so as to recover entire files later.
     *
     * Note that '\n' substitutes all other line breaks, such as "\r\n".
     */
    @Override
    public boolean next(BlockwiseTextWritable key, Text value) throws IOException {
        key.fileName = path.getName();
        key.offset = pos;
        value.clear();

        if (pos >= end) {
            return false;
        }

        Text blockContent = new Text();
        Text line = new Text();

        while (pos < end) {
            pos += reader.readLine(line);
            blockContent.append(line.getBytes(), 0, line.getLength());
            blockContent.append(LFs, 0, LFs.length);
        }

        if (totalLength < blockContent.getLength()) {
            value.set(blockContent.getBytes(), 0, totalLength);
        } else {
            value.set(blockContent.getBytes());
        }

        return true;
    }

{% endcodeblock %}

<!--more-->

## LDA best practice:

I think there are two common ways to use LDA in practice. First is the use in experimental condition, say, you have a bunch of small files on your disk. Then you want to upload them into HDFS, and call LDA in Spark. This is a usual way if you just want to do some experiments with LDA. In other words, it is an off-line training process. The second way of using LDA is an industrial use. You may have a streaming pipe, which feeds new data from Twitter or some other websites into your system. You may choose to put those data into a distributed storage such as HDFS or HBase, or you just process the data stream.

Think them boldly, they are totally different. With respect to the usage of LDA, we should take care of the two scenarios simultaneously. Both of them are useful so we should not give up each of them.

## Offline scenario of LDA

In the offline scenario, maybe you are not responsible for pre-processing. Instead, you just leave it to the end-user. Users transform the raw texts into the format you specify, and upload them into HDFS so your LDA application can read them directly. In this way, what we need to do is just specify the input format. What a relief !

Maybe you can help end-users one step more. You write a program, sequential or parallel, whichever is OK, to help the pre-processing for end-user. Just like what Mahout does. End-user may write an ugly shell program as coordinator, to control the overall workflow. In this way, you can write a program to transform the small files (raw texts) into a huge file which lines represents texts, with filenames in the front of the line plus a separator.

But, I think a better way is melding the pre-process with LDA. What the end-user does is just upload his raw texts on HDFS. In this way, we must provide the function to read all texts and their corresponding filenames in. Then we implement a `CombineFileInputFormat`, a `CombineFileRecordReader`, a `FileLineWritable` and an interface looks like `textFiles` to support the scenario.

{% codeblock Interface exposed to end-user - MLUtils.scala lang:scala %}

  /**
   * Reads a bunch of small files from HDFS, or a local file system (available on all nodes), or any
   * Hadoop-supported file system URI, and return an RDD[(String, String)].
   *
   * @param path The directory you should specified, such as
   *             hdfs://[address]:[port]/[dir]
   *
   * @param minSplits Suggested of minimum split number
   *
   * @return RDD[(fileName: String, content: String)]
   *         i.e. the first is the file name of a file, the second one is its content.
   */
  def smallTextFiles(sc: SparkContext, path: String, minSplits: Int): RDD[(String, String)] = {
    val fileBlocks = sc.hadoopFile(
      path,
      classOf[BatchFileInputFormat],
      classOf[BlockwiseFileKey],
      classOf[BytesWritable],
      minSplits)

    fileBlocks.mapPartitions { iterator =>
      var lastFileName = ""
      val mergedContents = ArrayBuffer.empty[(String, Text)]

      for ((block, content) <- iterator) {
        if (block.fileName != lastFileName) {
          mergedContents.append((block.fileName, new Text()))
          lastFileName = block.fileName
        }

        mergedContents.last._2.append(content.getBytes, 0, content.getLength)
      }

      mergedContents.map { case (fileName, content) =>
        (fileName, content.toString)
      }.iterator
    }
  }

{% endcodeblock %}

I am not mean that it's the best practice. Indeed, it is very bad to put lots of small files on HDFS, for it will occupy so many index entries than bad performance will occur. I just talk about one feasible way. However, there are some tangle problems we must solve.

First of all is the block size of your HDFS. Although we mean "small files", but how small it is? Will its size larger than a single block in HDFS? The answer is Yes, it is possible. So we must handle the joint of blocks for each file, especially when the file is a multi-byte one, say, UTF encoded. Characters on the edge of blocks will be separated into two parts. We must take the responsibility to merge them together seamlessly.

The key point is, we do not like shuffle, especially the unnecessary one. We hope that blocks of each file could be stay in the same node, so that we can merge them together without shuffle. As with the [blog](http://www.idryman.org/blog/2013/09/22/process-small-files-on-hadoop-using-combinefileinputformat-1/) says, if we override the `isSplitable()` function and set the return value to false, then we can keep a single file in the same `split`. `HadoopRDD` treats a `split` as a single partition. If so, we just need to merge blocks of a single file in one partition without any shuffle. Very happy!

However, we find that what the blog says is wrong. The `isSplitable()` is really useful, but just for `FileInputFormat`, which is the parent class of `CombineFileInputFormat`. The latter has a very complicated logic to, you know, divide blocks into splits. However, in order to improve the read performance, `CombineFileInputFormat` constructs a node list and a rack list, to chain blocks in the same node, or the same rack together, and send them into the same split. Remember that HDFS has replications. If there are 3 replicas of a block, then `CombineFileInputFormat` could have the possibility to read any of the 3 replications, ~~so we cannot ensure that the class will read any blocks from any nodes! It seems that a global shuffle is inevitable.~~

Funny, we also find that there is a class named `MultiFileInputFormat`, which is the predecessor of `CombineFileInputFormat`. It is deprecated now, because of the low efficiency due to unawareness of data locality (per node / rack).

~~There must be some trade-offs here. We should solve the shuffle in HDFS level, or we need solve the shuffle in Spark level. If sizes of all files are smaller than the block size, there is nothing hard to solve. But we cannot assume things like that.~~

## Online scenario of LDA

Now let's turn into another direction. Note that a online product will never take the way described above. You know, people comes from data process division would not be silly to save all raw texts in local disks, then upload it to servers when processing. Massive data, in my opinion, should be stay in an appropriate place. In this scenario, raw texts or web page should be stored in a KV store, such as HBase (facebook has a nice [paper](http://research.cs.wisc.edu/adsl/Publications/fbmessages-fast14.pdf) talking about the performance issues of HBase atop of HDFS). Small texts or articles should be treated same with pictures from websites. So, in reality, HBase will be used. However, Spark has no external storage except HDFS and local disk. So I think it is the time to add new storage. 
 
## In case of using a our own customized partitioner

Ah... It's awful! You know what, one of the reasons than Spark beats its counterparts is customized partitioners. It is the first time that you can arrange/rearrange your items according to your wishes such easily. One idea to settle our problem is using a customized partitioner to rearrange our KV pairs. However, new partitioner is useful only when we do join of two RDDs iteratively, such as `PageRank`. But if we only need to shuffle things once time, it will be helplessness, for you cannot avoid the first-time shuffle.

## Where the shuffle from? How to do trade-off?

We talked about needless shuffle just now. So question is, where is the shuffle from, and how to do trade-off? First, huge-files (or small files with smaller block size) could be cut off due to the fixed block size, so we want blocks belong to a single file could stay in the same split (partition, in dialect of Spark), then we can combine them together to recover the single file without any shuffle. The process is essential, because there will be multi-bytes characters such as UTF could be split. 

Second, `CombineFileInputFormat` cannot preserve the property for us, due to the consideration of efficiency (see `CombineFileInputFormat` and `MultiFileInputFormat` as an example). Mostly because of the replication in HDFS, the fault tolerance function in HDFS, blocks of a single file could be read at any nodes. 

So there are the trade-offs between "shuffle HDFS level" with "shuffle Spark level", and between "efficiency when reading blocks" with "efficiency due to shuffle-free", and eventually between "efficiency" with "security".

## Take a deep breath - Full disclosure of locality in Hadoop

To grasp the secret of locality of Hadoop IO, I had to dive deep into `InputFormat` code in `mapred`. Due to the use of Spark, I choose `FileInputFormat` as the breach. First you should keep these concepts in mind, which will be used commonly later. They are *rack*, *node*, *file*, *block* and *replica*. A rack is composed of several nodes, nodes are machines composing HDFS in Hadoop. A file is composed of several blocks. A block could have several replicas, by default 3. Note that your Hadoop workers could cover all HDFS nodes, but there could also mismatch between Hadoop workers and HDFS nodes. Note also that replicas of a block are usually span different racks, due to the consideration of robustness.

Things could be a little bit more complicated, if we add the workers of Hadoop in. Program could span across different workers, data could span across different nodes. So, question is, how to arrange the mapping of programs in each worker and blocks in each node, to get the best locaility, i.e. the less network communication when reading files on HDFS?

This is not easy, since there are many layers between program with block. Program is aware of file directly. File divided into several blocks. Block could be located in each nodes, and its replicas could be located in any other nodes. Different nodes could in different racks. Let's begin from our program. Take Spark as an example, you may call `hadoopRDD = sc.textFile(path)` to tell Spark read a file in. The path could be a local disk path, or more commonly, a HDFS path. `hadoopRDD` is usually partitioned for distributed computing. So, where is the partition information from? The answer is `Split` in HDFS. `Split`, or more specifically, `FileSplit`, which is used in `FileInputFormat`. `FileSplit` is an approach to arrange the mapping of **blocks and programs**.

Each `FileSplit` is a block set, in which blocks will be computed in the same worker, i.e. they are partitioned together. To preserve the locaility, `FileSplit` takes lots of efforts to place appropriate blocks together. Such as contribution computing, and node <-> block, rack <-> block double linked lists, etc. Note that shuffle in Spark is only related to the `Split`, because `Split` serves as a layer to shield the details in HDFS, which means that, if and only if we put blocks of an entire file into the same `Split`, our Spark is then "shuffle-free". But we cannot arrange different small files into a split in an random order, because different small files could be in everywhere on the HDFS cluster. If we put two files which are far apart in the same `Split`, bad performance will occur. Here we degenerate the program - block mapping to **split - block** mapping.

### Node/Rack contribution computing

We should remind ourselves that there is little chance to put blocks in the same node in the same `Split`, because we cannot directly access blocks, instead we just specify the file path. Suppose that we have a `Split`, in which there are 3 blocks, and come from 8 nodes. 8 nodes belong to 4 racks. Moreover, each block has 3 replicas in total. Let's assume that the lengths of 3 blocks are 100, 150, 75, respectively. How to arrange the `perferedLocation` in this scenario? Namely, on which workers should the `Split` be processed?

![pic-1](/images/2014/03/pic-1.png)

First of all, we all agree that the `preferedLocation` should be a subset of all nodes of our block. In our example, it would be a subset of [h1 ... h8]. The second is, how to sort the subset, so as to make "the best" node at first, then "the second-best" one, ... 

There are two different ways to arrange the `preferedLocation`- the rack-aware way and the rack-free way. Let's first decide what is the criteria of sort, i.e. what kind of node is "the best"? As illustrated in the picture above, we define a concept named "effective size". Effective size of a node is how many effective bytes of data of the split on this node. Effective size of a rack is how many effective bytes of data of the split on this rack. What effective bytes means is distinguishing block size. Say, Rack4 has two blocks, each block's size is 75. But the effective size of Rack4 is not 150, it is still 75, because the two blocks are the same - they are replicas.

The rack-aware way is that we treate rack the same important with node. After we get the effective size, we can give them an order as below:

1. Rack 2 (250)
    1. h4 (150)
    2. h3 (100)
2. Rack 1 (175)
    1. h1 (175)
    2. h2 (100)
3. Rack 3 (150)
    1. h5 (150)
    2. h6 (150)
4. Rack 4 (75)
    1. h7 (75)
    2. h8 (75)

So the priority order is **h4 > h3 > h1 > h2 > h5 > h6 > h7 > h8**.

In the other way, the rack-free way is simple. It just ignore the rack information, and sorts nodes via effective bytes of nodes:

1. h1 (175)
2. h4 (150)
3. h5 (150)
4. h6 (150)
5. h2 (100)
6. h3 (100)
7. h7 (75)
8. h8 (75)

Then the order is **h1 > h4 > h5 > h6 > h2 > h3 > h7 > h8**.

For more details, see this [test code](https://github.com/apache/hadoop-common/blob/release-1.0.4/src/test/org/apache/hadoop/mapred/TestGetSplitHosts.java).

### Double linked lists

`CombineFileInputFormat` chooses another way to keep locaility. It uses double linked list to chain blocks together, then sweep the chain per node, then per rack, to generate locaility-preserved split. This is cool if all small files are smaller than one block size. But if there is file content span across two blocks or more, especially the content has UTF8 code, it will get worse.

{% codeblock Double linked lists sweep for constructing split - CombineFileInputFormat.java lang:java%}

  /**
   * Return all the splits in the specified set of paths
   */
  private void getMoreSplits(JobConf job, Path[] paths, 
                             long maxSize, long minSizeNode, long minSizeRack,
                             List<CombineFileSplit> splits)
    throws IOException {

    // all blocks for all the files in input set
    OneFileInfo[] files;    
  
    // mapping from a rack name to the list of blocks it has
    HashMap<String, List<OneBlockInfo>> rackToBlocks = 
                              new HashMap<String, List<OneBlockInfo>>();

    // mapping from a block to the nodes on which it has replicas
    HashMap<OneBlockInfo, String[]> blockToNodes = 
                              new HashMap<OneBlockInfo, String[]>();

    // mapping from a node to the list of blocks that it contains
    HashMap<String, List<OneBlockInfo>> nodeToBlocks = 
                              new HashMap<String, List<OneBlockInfo>>();

    ...
    
    // process all nodes and create splits that are local
    // to a node. 
    for (Iterator<Map.Entry<String, 
         List<OneBlockInfo>>> iter = nodeToBlocks.entrySet().iterator(); 
         iter.hasNext();) {

      Map.Entry<String, List<OneBlockInfo>> one = iter.next();
      nodes.add(one.getKey());
      List<OneBlockInfo> blocksInNode = one.getValue();

      // for each block, copy it into validBlocks. Delete it from 
      // blockToNodes so that the same block does not appear in 
      // two different splits.
      for (OneBlockInfo oneblock : blocksInNode) {
        if (blockToNodes.containsKey(oneblock)) {
          validBlocks.add(oneblock);
          blockToNodes.remove(oneblock);
          curSplitSize += oneblock.length;

          // if the accumulated split size exceeds the maximum, then 
          // create this split.
          if (maxSize != 0 && curSplitSize >= maxSize) {
            // create an input split and add it to the splits array
            addCreatedSplit(job, splits, nodes, validBlocks);
            curSplitSize = 0;
            validBlocks.clear();
          }
        }
      }
      // if there were any blocks left over and their combined size is
      // larger than minSplitNode, then combine them into one split.
      // Otherwise add them back to the unprocessed pool. It is likely 
      // that they will be combined with other blocks from the same rack later on.
      if (minSizeNode != 0 && curSplitSize >= minSizeNode) {
        // create an input split and add it to the splits array
        addCreatedSplit(job, splits, nodes, validBlocks);
      } else {
        for (OneBlockInfo oneblock : validBlocks) {
          blockToNodes.put(oneblock, oneblock.hosts);
        }
      }
      validBlocks.clear();
      nodes.clear();
      curSplitSize = 0;
    }

    ...
    
{% endcodeblock %}

### How about read?

After the discussion above, we know how MapReduce program keep locaility when composing `Split` with blocks. We are very happy with the sorted `perferedLocation`, and send it back to partitions on Spark. The next step is Spark framework launchs executors on workers according to the `perferedLocation`, say, h4 WRT the example above. The launched executor on h4 read these blocks in the split now. But, how does h4 know which nodes to fetch each block? Remeber that each block has 3 replicas!

Begining from `RecordReader` we can reveal the process of reading. Let's take our `BatchFileRecordReader` as an example.

{% codeblock Constructer of BatchFileRecoderReader - BatchFileRecorderReader.java lang:java%}

    public BatchFileRecordReader(
            CombineFileSplit split,
            Configuration conf,
            Reporter reporter,
            Integer index)
            throws IOException {
        path = split.getPath(index);
        startOffset = split.getOffset(index);
        pos = startOffset;
        end = startOffset + split.getLength(index);

        FileSystem fs = path.getFileSystem(conf);
        fileIn = fs.open(path);
        fileIn.seek(startOffset);

        totalMemory = Runtime.getRuntime().totalMemory();
    }

{% endcodeblock %}

In the code above, we get `path` from `split`, which represents the current file path (Note! It is not the block path.). Then we can get a `fileIn` which is actually a `FSDataInputStream`. We then `seek` it to the `startOffset` of our block. Wait for a second, we do not use `perferedLocation` in `split` at all! It is strange, we took lots of efforts just now, but it is not used here.

We should remember here that the `split` is just used for providing a computing place for these set of blocks. Only so much. Reading is controlled by other code. Let's go into the `FSDataInputStream`. However, there is really nothing, just some useless-like code as below:

{% codeblock FSDataInputStream.java lang:java%}

public class FSDataInputStream extends DataInputStream
    implements Seekable, PositionedReadable, Closeable {

    public FSDataInputStream(InputStream in)
        throws IOException {
        super(in);
        if( !(in instanceof Seekable) || !(in instanceof PositionedReadable) ) {
            throw new IllegalArgumentException(
            "In is not an instance of Seekable or PositionedReadable");
        }
    }
    
    public synchronized void seek(long desired) throws IOException {
        ((Seekable)in).seek(desired);
    }
    ...
}

{% endcodeblock %}

OK, let's force from another way. Note that `fileIn` is return by calling `fs.open()`. `fs` here is usually `DistributedFileSystem`. Then we find that `DistributedFileSystem` just wraps a `DFSInputStream` to `FSDataInputStream`. The former is implemented in `DFSClient`. Our expected function in `DFSInputStream` is `blockSeekTo()`, which is in charge of finding an appropriate block given offset. Then it will find the best DataNode, and read data from it.

{% codeblock Find an appropriate block and select a DataNode  - DFSClient.java lang:java%}

    DatanodeInfo chosenNode = null;
    int refetchToken = 1; // only need to get a new access token once
    while (true) {
        //
        // Compute desired block
        //
        LocatedBlock targetBlock = getBlockAt(target, true);
        assert (target==this.pos) : "Wrong postion " + pos + " expect " + target;
        long offsetIntoBlock = target - targetBlock.getStartOffset();

        DNAddrPair retval = chooseDataNode(targetBlock);
        chosenNode = retval.info;
        InetSocketAddress targetAddr = retval.addr;
        ...
    }

{% endcodeblock %}

The most important function here is `chooseDataNode()`. It is very simple, just select the first DataNode in its DataNode list. If the first one is unreachable, it will try to connect to the second one, and so on. The comments in `bestNode()` function mentioned that DataNode list has already sorted in the priority order. It is strange that when it is sorted?

Indeed, the block priority order is set when the file is open. See `openInfo()`, it calls `callGetBlockLocations()` to set the order. The latter query information from `NameNode`, in `getBlockLocations()`:

{% codeblock Get block locations and sorted in the priority order  - FSNamesystem.java lang:java%}

    LocatedBlocks getBlockLocations(String clientMachine, String src,
        long offset, long length) throws IOException {
        LocatedBlocks blocks = getBlockLocations(src, offset, length, true, true);
        if (blocks != null) {
            //sort the blocks
            DatanodeDescriptor client = host2DataNodeMap.getDatanodeByHost(
                clientMachine);
            for (LocatedBlock b : blocks.getLocatedBlocks()) {
                clusterMap.pseudoSortByDistance(client, b.getLocations());
            }
        }
        return blocks;
    }

{% endcodeblock %}

We can see that it calls `pseudoSortByDistance()` of `clusterMap` to sort according to the distance. Untill now, we get the full picture of how HDFS keep locaility for applications.

## New Design and Implementation

## Interesting test code




key ideas:

- ~~癫狂小文件：由small files input探索HDFS：功用、功效、策略~~

- ~~Spark的处理方式：partition自己定制，把O(n^2)的shuffle变成O(n)的？~~

- Hbase作为替代

- ~~Hdfs combine file与multi file对比，后者为什么被deprecated~~

- ~~Shuffle为什么存在，如何避免，是在底层（HDFS）还是在上层（Spark应用）？~~

- ~~偶尔要为replication容错付出更多代价~~

- ~~Combine file input format，blog上错误多影响广，以及不友好的API，糟糕的示例和注释。~~

- ~~实际场景？学术研究VS线上应用。什么才是LDA输入的最佳实践？Mahout的处理方式？~~

- 文件系统的设计思考，可引入我自己的文件系统。

- 代码的简洁如何保证？同时如何保证性能？隐藏在简洁代码中的黑魔法。

- Partition，split，以及种种locaility preserve的思考（为什么程序性能这么差？1024个partition从何而来？）

- Google在bigtable和GFS的思考

- LineRecorder中的readline函数如何实现越过block向后看？
