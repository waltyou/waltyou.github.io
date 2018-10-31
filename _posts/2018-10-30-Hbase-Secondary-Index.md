---
layout: post
title: Hbase RegionServer Size的调整
date: 2018-10-30 14:04:04
author: admin
comments: true
categories: [Hbase]
tags: [Big Data, Hbase]
---

有时候表的 rowkey 设计，并不能满足我们的查询，这时候就要使用“二级索引”。

<!-- more -->

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 二级索引和备用查询方式

这个一部分也可以被叫做：“虽然我的rowkey 被这样子设计了，但是我想那样子去搜索。”

比如 rowkey 形如 user-timestamp， 但是我们想找出在一定时间范围内有哪些活动的用户。
当然，用user去搜索很简单，因为它位于 rowkey 开头。

关于处理这个问题的最佳方法没有一个答案，因为它取决于：
- user 数量
- 数据大小和数据到达率
- 报告要求的灵活性（例如，完全临时日期选择与预配置范围）
- 期望的查询执行速度（例如，对于某些特定报告，90秒可能是合理的，而对于其他人来说可能太长）

并且解决方案也受到群集大小以及解决方案需要多少处理能力的影响。
常用技术在下面的小节中。
这是一个全面但并非详尽无遗的方法清单。

二级索引需要额外的集群空间和处理。这正是RDBMS中发生的情况，因为创建备用索引的行为需要空间和处理周期来更新。
在这方面，RDBMS产品更先进，可以处理开箱即用的替代索引管理。
但是，HBase在更大的数据量下可以更好地扩展，因此这是一项功能权衡。

在实施任何这些方法时，请注意[Apache HBase性能调优](http://hbase.apache.org/book.html#performance)。

## 1. 过滤查询（Filter Query）

根据具体情况，使用[客户端请求过滤器](http://hbase.apache.org/book.html#client.filter)可能是合适的。
在这种情况下，不会创建二级索引。
但是，不要尝试从应用程序（即单线程客户端）对这样的大型表进行全扫描。

## 2. 定期更新的二级索引（Periodic-Update Secondary Index）

可以把二级索引建在另一个表中，该表通过MapReduce作业定期更新。
该作业可以在日内执行，但根据加载策略，它仍可能与主数据表不同步。

### 官方例子

以下是使用HBase作为源和MapReduce的接收器的示例。此示例将简单地将数据从一个表复制到另一个表。
```java
Configuration config = HBaseConfiguration.create();
Job job = new Job(config,"ExampleReadWrite");
job.setJarByClass(MyReadWriteJob.class);    // class that contains mapper

Scan scan = new Scan();
scan.setCaching(500);        // 1 is the default in Scan, which will be bad for MapReduce jobs
scan.setCacheBlocks(false);  // don't set to true for MR jobs
// set other scan attrs

TableMapReduceUtil.initTableMapperJob(
  sourceTable,      // input table
  scan,             // Scan instance to control CF and attribute selection
  MyMapper.class,   // mapper class
  null,             // mapper output key
  null,             // mapper output value
  job);
TableMapReduceUtil.initTableReducerJob(
  targetTable,      // output table
  null,             // reducer class
  job);
job.setNumReduceTasks(0);

boolean b = job.waitForCompletion(true);
if (!b) {
    throw new IOException("error with job!");
}
```

这一步需要了解一下 TableMapReduceUtil 是做什么的。
它会把 TableOutputFormat 当作这个 job 的 outputFomrat 类，同时在 config 里面定义一些参数，比如TableOutputFormat.OUTPUT_TABLE。
同时也把 Reducer 输出的 key 类型定位 ImmutableBytesWritable ， value 类型定为 Writable。
当然这些也可以有程序员自己设定，不过用 TableMapReduceUtil 更简单。

以下是示例 Mapper ，它将创建一个Put并匹配输入Result并将其发出。注意：这是CopyTable实用程序的功能。

```java
public static class MyMapper extends TableMapper<ImmutableBytesWritable, Put>  {

  public void map(ImmutableBytesWritable row, Result value, Context context) throws IOException, InterruptedException {
    // this example is just copying the data from the source table...
      context.write(row, resultToPut(row,value));
    }

    private static Put resultToPut(ImmutableBytesWritable key, Result result) throws IOException {
      Put put = new Put(key.get());
      for (KeyValue kv : result.raw()) {
        put.add(kv);
      }
      return put;
    }
}
```
实际上没有reducer步骤，因此TableOutputFormat负责将Put发送到目标表。 

这只是一个示例，开发人员可以选择不使用TableOutputFormat并自己连接到目标表。

[完整链接](http://hbase.apache.org/book.html#mapreduce.example.readwrite)。

## 3. 双写二级索引（Dual-Write Secondary Index）

另一种策略是在向集群发布数据时构建二级索引（例如，写入数据表，写入索引表）。

如果在数据表已经存在之后，采用这种方法，则需要使用MapReduce作业对二级索引进行初始化。
可以参考上面第二点。

## 4. 汇总表（Summary Tables）

如果时间范围非常广（例如，一年的报告）并且数据量很大，则汇总表是一种常用方法。
这些将通过MapReduce作业生成到另一个表中。

### 官方例子

此示例将计算表中值的不同实例的数量，并将这些汇总计数写入另一个表中。

```java
Configuration config = HBaseConfiguration.create();
Job job = new Job(config,"ExampleSummary");
job.setJarByClass(MySummaryJob.class);     // class that contains mapper and reducer

Scan scan = new Scan();
scan.setCaching(500);        // 1 is the default in Scan, which will be bad for MapReduce jobs
scan.setCacheBlocks(false);  // don't set to true for MR jobs
// set other scan attrs

TableMapReduceUtil.initTableMapperJob(
  sourceTable,        // input table
  scan,               // Scan instance to control CF and attribute selection
  MyMapper.class,     // mapper class
  Text.class,         // mapper output key
  IntWritable.class,  // mapper output value
  job);
TableMapReduceUtil.initTableReducerJob(
  targetTable,        // output table
  MyTableReducer.class,    // reducer class
  job);
job.setNumReduceTasks(1);   // at least one, adjust as required

boolean b = job.waitForCompletion(true);
if (!b) {
  throw new IOException("error with job!");
}
```

在此示例 Mapper 中，选择具有String-value的列作为要汇总的值。
此值用作从 Mapper 发出的键，IntWritable表示实例计数器。

```java
public static class MyMapper extends TableMapper<Text, IntWritable>  {
  public static final byte[] CF = "cf".getBytes();
  public static final byte[] ATTR1 = "attr1".getBytes();

  private final IntWritable ONE = new IntWritable(1);
  private Text text = new Text();

  public void map(ImmutableBytesWritable row, Result value, Context context) throws IOException, InterruptedException {
    String val = new String(value.getValue(CF, ATTR1));
    text.set(val);     // we can only emit Writables...
    context.write(text, ONE);
  }
}
```

在reducer中，计算“ones”（就像执行此操作的任何其他MR示例一样），然后发出Put。

```java
public static class MyTableReducer extends TableReducer<Text, IntWritable, ImmutableBytesWritable>  {
  public static final byte[] CF = "cf".getBytes();
  public static final byte[] COUNT = "count".getBytes();

  public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
    int i = 0;
    for (IntWritable val : values) {
      i += val.get();
    }
    Put put = new Put(Bytes.toBytes(key.toString()));
    put.add(CF, COUNT, Bytes.toBytes(i));

    context.write(null, put);
  }
}
```

[完整链接](http://hbase.apache.org/book.html#mapreduce.example.summary)

## 5. 协处理器二级索引（Coprocessor Secondary Index）

协处理器就像RDBMS触发器一样。这些是在0.92中添加的。

后续添加详细介绍，现在可以看[官方文档](http://hbase.apache.org/book.html#cp)


---

# 参考

1. [Apache HBase ™ Reference Guide](http://hbase.apache.org/book.html) 

