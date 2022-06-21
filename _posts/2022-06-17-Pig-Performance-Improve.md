---
layout: post
title:  一个Pig 调优的例子
date: 2022-06-17 18:11:04
author: admin
comments: true
categories: [Pig]
tags: [Big Data, Pig]
---
<!-- more -->

---

* 目录
  {:toc}

---

## 背景

工作中一个daily run的 Pig 脚本在繁忙集群中失败率很高（>50%），自己不得不经常重启这个失败的job，浪费了很多时间精力，所以这次试着解决一下它。

经过调查研究后，最后从两个方面进行调优，一个是优化处理逻辑（避免重复计算、分而治之)，另一个是实现可以触发combiner的UDF（[pig combiner](https://pig.apache.org/docs/latest/perf.html#combiner))。失败率从53% 降为了 0%， 运行速度缩短了 44%左右。

## 分析问题

首先理清 PIG 是在哪一步失败的。

这可以通过日志看出来，pig会将整个pig script划分出多个MapReduce job，并且在日志中打印出每个job对应计算的aliases，类似下面第三行显示的那样子：

```plaintext
2022-06-15 07:13:26,387 [SimpleAsyncTaskExecutor-1]  INFO executionengine.mapReduceLayer.MapReduceLauncher - HadoopJobId: job_1644264078244_274211
2022-06-15 07:13:26,388 [SimpleAsyncTaskExecutor-1]  INFO executionengine.mapReduceLayer.MapReduceLauncher - Processing aliases txn_addr_g,txn_addr_group,txn_addr_j,txn_addr_prune
2022-06-15 07:13:26,388 [SimpleAsyncTaskExecutor-1]  INFO executionengine.mapReduceLayer.MapReduceLauncher - detailed locations: M: txn_addr_j[89,13],txn_addr_j[89,13] C:  R: txn_addr_group[95,17],txn_addr_g[97,13],txn_addr_prune[100,17]
```

可以看出 job_1644264078244_274211， 它会计算txn_addr_g, txn_addr_group, txn_addr_j, txn_addr_prune 这四个aliases，并且也列出来MapReduce各个阶段产生的alias。

> map 阶段是 M: txn_addr_j[89,13],txn_addr_j[89,13]
>
> combiner阶段没有 C:
>
> reduce阶段是：R: txn_addr_group[95,17],txn_addr_g[97,13],txn_addr_prune[100,17]

找到容易失败的job后，根据日志，着重分析一下相关alias的计算逻辑，用pig语言简单描述如下：

* load 表 a (name, country, update_ts)
* load 表 b (id, name, ts)
* join_table = join b by name, a by name;
* group_table = group join_table by id;
* final = foreach group_table {
  filter_t = FILTER join_table BY update_ts <= ts;
  order_t = order filter_t BY  update_ts DESC;
  limit_t = LIMIT order_t 1;
  GENERATE FLATTEN(limit_t);
  };

用 SQL 描述一下：

* a = select name, country, update_ts from tableA;
* b = select id, name, ts from tableB;
* join_table = select b.id, b.name, b.ts, a.country, a.update_ts from a join b on a.name = b.name;
* filter_table = select * from join_table where update_ts <= ts;
* order_t = select *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY update_ts DESC) AS row from filter_table;
* final = select id, name. ts, country, update_ts from order_t where row = 1;

表a 是一张按天分区的表，同一个name的数据可能在不同分区下很多条，另外它的时间跨度有8年之久，整张表数据大小有2.4T，并且name 列的skew较为严重。
表b是一张小表，大概2G左右，它的ts都集中在具体的某一天。

job经常失败在两个地方，一个是两张表join的时候，因为join key "name"的分布不均匀。另外一个地方是group时，在经历上面join后，表a字段name的不均匀会传导到 join_table 的id 字段，这样子我们在group时候，也会产生skew。

这些分布不均匀的name或者id，会导致MapReduce 过程中，大部分的reduce task 可以在短时间内完成，而又个别几个reduce task 会跑的特别久，直到失败，从而让整个application 失败。

## 解决思路

问题找了出来，那么该如何解决呢？简单的思路就是让reduce task接收到的数据少一些、均匀一些。

经过网上搜索工程之后，发现一些有用的资料：

* [官方文档：Performance and Efficiency](https://pig.apache.org/docs/latest/perf.html)
* [Pig性能优化](https://www.cnblogs.com/kemaswill/p/3226754.html)

通过以上文档可以了解到：

* 对于join的失败，可以尝试 Specialized Joins
* 对于group的失败，可以尝试 combiner

### 1. Specialized Joins

在尝试 Replicated Joins 和 Skewed Joins 后，效果都不好。

Replicated Joins 的基本思路是想小表放入内存中，直接与大表的各个map进行join（类似Spark中Broadcast Hash Join），但是小表的大小不能超过pig.join.replicated.max.bytes=1G，因为在Pig实验过程中发现100M的数据会占用1G的内存（10倍）。然而我这个例子中，小表的大小大约在1.5G左右，在尝试调大 pig.join.replicated.max.bytes 为2G后，job失败于 OOM。此路不通。

Skewed Joins 的基本思路是对skew的key进行抽样，然后列出直方图，来决定如何分配key到不同的reducer上。有点类似 Spark 3.0 中的 [Dynamically Coalesce Shuffle Partitions](https://blog.cloudera.com/how-does-apache-spark-3-0-increase-the-performance-of-your-sql-workloads/) 然而实际尝试后，效果不佳，某些reducer task依然长时间跑不完。

看起来调整join的思路是走不通了。

### 2. 启用combiner

combiner 可以对map的输出结果就进行combine，从而使进入reducer的数据量大大减少，让性能大大提高，按照官方文档的说法就是 “magnitude improvement in performance"。

虽然combiner看起来好处多多，但是Pig启用它却有限制条件，官方原文为”The combiner is generally used in the case of non-nested foreach where all projections are either expressions on the group column or expressions on algebraic UDFs."

简单来讲，有以下几点：

* non-nested foreach： 就是说不能嵌套foreach
* 语句返回的结果只能有两种， expressions on the group （group所用到的key或者keys），或者是 algebraic UDF

用 Pig 语句来描述就是：

```pig
C = foreach B generate algebraic_function(xxx), ...., group.xxx;
```

所以如果使用一个 algebraic UDF 重写原始逻辑的话(省略了load 表a、表b的语句)，就会如下所示：

```pig
join_table = join b by name, a by name;
group_table = group join_table by id;
final = foreach group_table generate flatten(getLatestRecord(join_table));
```

`getLatestRecord` 里面实现的基础逻辑就是：

```pig
filter_t = FILTER join_table BY update_ts <= ts;
order_t = order filter_t BY update_ts DESC;
limit_t = LIMIT order_t 1;
```

通过官方文档[Make Your UDFs Algebraic](https://pig.apache.org/docs/latest/perf.html#algebraic-interface)，可以得知，只需要继承 algebraic 接口就行：

```java
public interface Algebraic{
    public String getInitial();
    public String getIntermed();
    public String getFinal();
}
```

这三个接口函数 `getInitial`, `getIntermed`, `getFinal` 的返回值都是一个类名，这些类都需要继承 `EvalFunc` 接口, 并实现 `exec` 方法。参考[官方文档](https://pig.apache.org/docs/latest/udf.html#algebraic-interface)，描述一下在MapReduce世界中这三个函数的区别：

| 函数名          | 调用位置 | input类型                   | output 类型   |
| --------------- | -------- | --------------------------- | ------------- |
| `getInitial`  | map 端   | List[input row]             | 自定义        |
| `getIntermed` | map 端   | List[`getInitial`的输出]  | 自定义        |
| `getFinal`    | reduce端 | List[`getIntermed`的输出] | UDF的输出类型 |

那么函数 myAlgebraicUdf 的实现就类似如下code：

```java
public class GetLatestRecord extends EvalFunc<Tuple> implements Algebraic {
    @Override
    public Tuple exec(Tuple input) throws IOException {
        return latest(input);
    }

    public String getInitial() {return Initial.class.getName();}
    public String getIntermed() {return Intermed.class.getName();}
    public String getFinal() {return Final.class.getName();}
    static public class Initial extends EvalFunc<Tuple> {
        public Tuple exec(Tuple input) throws IOException {return latest(input);}
    }
    static public class Intermed extends EvalFunc<Tuple> {
        public Tuple exec(Tuple input) throws IOException {return latest(input);}
    }
    static public class Final extends EvalFunc<Tuple> {
        public Tuple exec(Tuple input) throws IOException {return latest(input);}
    }
    static Tuple latest(Tuple input) throws ExecException {
        if (input == null || input.get(0) == null) {
            return null;
        }
        Tuple result = null;
        DataBag group = (DataBag) input.get(0);
        // item: b.id, b.name, b.ts, a.country, a.update_ts
        for (Tuple item : group) {
            Long update_ts = (Long) item.get(5);
            // FILTER join_table BY update_ts <= ts; 
            if(update_ts <= item.get(3)) {
              if (result == null) {
                  result = item;
              } else {
                  // order_t = order filter_t BY update_ts DESC;
                  // limit_t = LIMIT order_t 1;
                  if (update_ts > (int) result.get(5)) {
                      result = item;
                  } 
              }
            }
        }
        return result;
    }
}
```

### 3. 重排处理逻辑

经过测试，使用支持 Algebraic 的UDF后，group 的失败率大大降低。然而 join 这一步的skew问题还是没有解决。

阅读原始逻辑可以发现，join 是发生在 group 之前的，那么可不可以先group 再 join 呢？这样子可以在group 阶段先让数据量会大大减少，并且也可以使用 Algebraic UDF 加快速度和降低失败率。

答案是可以的。

原始逻辑中之所以要先 join 再 group，是因为在group 阶段需要使用 b 表中的 ts 字段。而我们已知表 b 的ts 都是一天之中的，所以我们可以把表 a 分为两部分处理。

第一部分是可以无视b表的ts，比如我们先加一个filter （update_ts <= min(ts)）从而获得一个数据集 a_1, 那么 a_1 里的数据所有 update_ts 都必然小于 join 后的 ts，这样子我们就可以放心大胆的直接group取最后的值，然后再与b表进行join。那么在join之前的数据量肯定大大减小（经测试可以减到 70G，相当于原先2.4T的 3%)，因为相当于通过group去了重。此外因为可以使用combiner，那么这次group + flatten的操作，其性能改善非常可观。

另外一部部分 a_2, 我们不得不还是只能先join，然后group，但是因为加了时间上的过滤（update_ts> min(ts)），我们只处理一天a表的数据，那么它的数据量就不大，我们还按照之前逻辑跑就可以了。

最后再将这两部分数据union起来，再group + flatten + getLatestRecord，就可以得到最终数据。在这一步，每个id 所能对应重复的数据最多就只会有两条，完美解决了skew的问题。

## 最终方案

### 1. 增加 Algebraic UDF

只需要重写一下上面 UDF `GetLatestRecord` 中的 `latest` 函数就行，因为不需要 b 表的 ts了，ts的过滤可以放到 PIG 语句中实现：

```java
static Tuple latest(Tuple input) throws ExecException {
    if (input == null || input.get(0) == null) {
        return null;
    }

    Tuple result = null;
    DataBag group = (DataBag) input.get(0);
    // item: a.name, a.country, a.update_ts , ....
    for (Tuple item : group) {
        if (result == null) {
            result = item;
        } else {
            Long update_ts = (Long) item.get(3);
            if (update_ts > (int) result.get(3)) {
                result = item;
            } 
        }
    }
    return result;
}
```

### 2. 改善后的处理逻辑

```pig
---- handle T-1
a_1 = filter a by ts <= $YesterDay_ts;
a_1_group = group a_1 by name;
a_1_snap = foreach a_1_group generate flatten(getLatestRecord(a_1));
-- each name only have one row in a_1_snap
a_1_join = join a_1_snap by name, b by name;

---- handle T
a_2 = filter a by ts > $YesterDay_ts;
-- a_2 only have one day data so it is small and easy to join
a_2_join = join a_2 by name, b by name;
a_2_group = group a_2_join by id;
a_2_snap = foreach a_2_group generate flatten(getLatestRecord(a_2_join));

---- Union T-1 & T and get final result
all = union a_1_join, a_2_snap;
group_table = group all by id;
final = foreach group_table generate flatten(getLatestRecord(all));
```

### 3. 效果

结果上述操作后，job速度提高了大约 44%，而且最重要的是失败率降从53% 降为了 0。

## 写在最后

调优要从代码逻辑入手，分析程序慢、卡、失败率高的具体原因是什么，然后对症下药，从不同角度试着解决问题。
