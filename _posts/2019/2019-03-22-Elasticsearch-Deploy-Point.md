---
layout: post
title: Elasticsearch 生产部署注意点
date: 2019-03-22 16:51:04
author: admin
comments: true
categories: [Elasticsearch ]
tags: [Big Data, Elasticsearch]
---

为了能在生产上，更好地使用 Elasticsearch，需要有些注意点，如硬件、配置等。

<!-- more -->

------




* 目录
{:toc}


------

# 生产部署前

## 1. 硬件

### 1）堆内存

Elasticsearch 默认安装后设置的堆内存是 1 GB。 对于任何一个业务部署来说， 这个设置都太小了。如果你正在使用这些默认堆内存配置，您的集群可能会出现问题。

这里有两种方式修改 Elasticsearch 的堆内存。最简单的一个方法就是指定 ES_HEAP_SIZE 环境变量。服务进程在启动时候会读取这个变量，并相应的设置堆的大小。 比如，你可以用下面的命令设置它：

```
export ES_HEAP_SIZE=10g
```
或者
```
./bin/elasticsearch -Xmx10g -Xms10g 
```

#### 非堆内存 （off-heap）：Lucene

标准的建议是把 50％ 的可用内存作为 Elasticsearch 的堆内存，保留剩下的 50％。当然它也不会被浪费，Lucene 会很乐意利用起余下的内存。

如果你不需要对分词字符串做聚合计算（例如，不需要 fielddata ）可以考虑降低堆内存。堆内存越小，Elasticsearch（更快的 GC）和 Lucene（更多的内存用于缓存）的性能越好。


#### 不要超过 32 GB！

JVM 在内存小于 32 GB 的时候会采用一个内存对象指针压缩技术。一旦你越过那个神奇的 32 GB 的边界，指针就会切回普通对象的指针。 每个对象的指针都变长了，就会使用更多的 CPU 内存带宽，也就是说你实际上失去了更多的内存。事实上，当内存到达 40–50 GB 的时候，有效内存才相当于使用内存对象指针压缩技术时候的 32 GB 内存。

这段描述的意思就是说：即便你有足够的内存，也尽量不要 超过 32 GB。因为它浪费了内存，降低了 CPU 的性能，还要让 GC 应对大内存。


#### 怎么设置堆内存？

确切的划分要根据 JVMs 和操作系统而定。 如果你想保证其安全可靠，设置堆内存为 31 GB 是一个安全的选择。 另外，你可以在你的 JVM 设置里添加 -XX:+PrintFlagsFinal 用来验证 JVM 的临界值， 并且检查 UseCompressedOops 的值是否为 true。对于你自己使用的 JVM 和操作系统，这将找到最合适的堆内存临界值。


#### Swapping 是性能的坟墓

最好的办法就是在你的操作系统中完全禁用 swap。这样可以暂时禁用：
```
sudo swapoff -a
```

如果需要永久禁用，你可能需要修改 /etc/fstab 文件，这要参考你的操作系统相关文档。

如果你并不打算完全禁用 swap，也可以选择降低 swappiness 的值。 这个值决定操作系统交换内存的频率。 这可以预防正常情况下发生交换，但仍允许操作系统在紧急情况下发生交换。

对于大部分Linux操作系统，可以在 sysctl 中这样配置：

```
vm.swappiness = 1
```
最后，如果上面的方法都不合适，你需要打开配置文件中的 mlockall 开关。 它的作用就是允许 JVM 锁住内存，禁止操作系统交换出去。在你的 elasticsearch.yml 文件中，设置如下：

```
bootstrap.mlockall: true
```

### 2）CPUs

大多数 Elasticsearch 部署往往对 CPU 要求不高。因此， 相对其它资源，具体配置多少个（CPU）不是那么关键。你应该选择具有多个内核的现代处理器，常见的集群使用两到八个核的机器。

如果你要在更快的 CPUs 和更多的核心之间选择，选择更多的核心更好。多个内核提供的额外并发远胜过稍微快一点点的时钟频率。

### 3）硬盘

如果你负担得起 SSD，它将远远超出任何旋转介质（注：机械硬盘，磁带等）。 基于 SSD 的节点，查询和索引性能都有提升。如果你负担得起，SSD 是一个好的选择。

如果你使用旋转介质，尝试获取尽可能快的硬盘（高性能服务器硬盘，15k RPM 驱动器）。

使用 RAID 0 是提高硬盘速度的有效途径，对机械硬盘和 SSD 来说都是如此。没有必要使用镜像或其它 RAID 变体，因为高可用已经通过 replicas 内建于 Elasticsearch 之中。

最后，避免使用网络附加存储（NAS）。人们常声称他们的 NAS 解决方案比本地驱动器更快更可靠。除却这些声称， 我们从没看到 NAS 能配得上它的大肆宣传。NAS 常常很慢，显露出更大的延时和更宽的平均延时方差，而且它是单点故障的。


### 4）Java 虚拟机

Java 8 强烈优先选择于 Java 7。不再支持 Java 6。Oracle 或者 OpenJDK 是可以接受的，它们在性能和稳定性也差不多。

最佳实践是客户端和服务器使用相同版本 JVM。

#### 请不要调整 JVM 设置

> JVM 暴露出几十个（甚至数百）的设置、参数和配置。 它们允许你微调 JVM 的几乎每一个方面。 当遇到一个旋钮，要打开它是人的本性。我们恳求你压制这个本性，而不要去调整 JVM 参数。Elasticsearch 是复杂的软件，并且我们根据多年的实际使用情况调整了当前 JVM 设置。 它很容易开始转动旋钮，并产生难以衡量的、未知的影响，并最终使集群进入一个缓慢的、不稳定的混乱的效果。当调试集群时，第一步往往是去除所有的自定义配置。多数情况下，仅此就可以恢复稳定和性能。


## 2. 重要配置的修改

sysctl -w vm.max_map_count=262144

/etc/sysctl.conf
vm.max_map_count = 262144

### 1）指定名字

你最好给你的生产环境的集群改个名字，改名字的目的很简单， 就是防止某人的笔记本电脑加入了集群这种意外。

可以在 elasticsearch.yml 中配置：

```
cluster.name: elasticsearch_production
```

我们建议给每个节点设置一个有意义的、清楚的、描述性的名字，同样你可以在 elasticsearch.yml 中配置：
```
node.name: elasticsearch_005_data
```

### 2）路径

最好的选择就是把你的数据目录配置到安装目录以外的地方， 同样你也可以选择转移你的插件和日志目录。

可以更改如下：

```yaml
path.data: /path/to/data1,/path/to/data2 

# Path to log files:
path.logs: /path/to/logs

# Path to where plugins are installed:
path.plugins: /path/to/plugins
​````
```

数据可以保存到多个不同的目录， 如果将每个目录分别挂载不同的硬盘，这可是一个简单且高效实现一个软磁盘阵列（ RAID 0 ）的办法。Elasticsearch 会自动把条带化（注：RAID 0 又称为 Stripe（条带化），在磁盘阵列中,数据是以条带的方式贯穿在磁盘阵列所有硬盘中的） 数据分隔到不同的目录，以便提高性能。

### 3）最小主节点数

`minimum_master_nodes` 设定对你的集群的稳定 *极其* 重要。 当你的集群中有两个 masters（注：主节点）的时候，这个配置有助于防止 *脑裂* ，一种两个主节点同时存在于一个集群的现象。

如果你的集群发生了脑裂，那么你的集群就会处在丢失数据的危险中，因为主节点被认为是这个集群的最高统治者，它决定了什么时候新的索引可以创建，分片是如何移动的等等。如果你有 *两个* masters 节点， 你的数据的完整性将得不到保证，因为你有两个节点认为他们有集群的控制权。

这个配置就是告诉 Elasticsearch 当没有足够 master 候选节点的时候，就不要进行 master 节点选举，等 master 候选节点足够了才进行选举。

此设置应该始终被配置为 master 候选节点的法定个数（大多数个）。法定个数就是 `( master 候选节点个数 / 2) + 1` 。 这里有几个例子：

- 如果你有 10 个节点（能保存数据，同时能成为 master），法定数就是 `6` 。
- 如果你有 3 个候选 master 节点，和 100 个 data 节点，法定数就是 `2` ，你只要数数那些可以做 master 的节点数就可以了。
- 如果你有两个节点，你遇到难题了。法定数当然是 `2` ，但是这意味着如果有一个节点挂掉，你整个集群就不可用了。 设置成 `1` 可以保证集群的功能，但是就无法保证集群脑裂了，像这样的情况，你最好至少保证有 3 个节点。

你可以在你的 `elasticsearch.yml` 文件中这样配置：

```yaml
discovery.zen.minimum_master_nodes: 2
```

基于这个原因， `minimum_master_nodes` （还有一些其它配置）允许通过 API 调用的方式动态进行配置。 当你的集群在线运行的时候，你可以这样修改配置：

```js
PUT /_cluster/settings
{
    "persistent" : {
        "discovery.zen.minimum_master_nodes" : 2
    }
}
```

### 4）集群恢复方面的配置

当需要为整个集群做离线维护，会出现批量节点退出、加入集群。

在整个过程中，你的节点会消耗磁盘和网络带宽，来回移动数据，因为没有更好的办法。对于有 TB 数据的大集群, 这种无用的数据传输需要 *很长时间* 。如果等待所有的节点重启好了，整个集群再上线，所有的本地的数据都不需要移动。

现在我们知道问题的所在了，我们可以修改一些设置来缓解它。 首先我们要给 ELasticsearch 一个严格的限制：

```yaml
gateway.recover_after_nodes: 8
```

保证 Elasticsearch 在存在 8 个节点以上（数据节点或者 master 节点）再进行数据恢复。 这个值的设定取决于个人喜好：整个集群提供服务之前你希望有多少个节点在线？这种情况下，我们设置为 8，这意味着至少要有 8 个节点，该集群才可用。

现在我们要告诉 Elasticsearch 集群中 *应该* 有多少个节点，以及我们愿意为这些节点等待多长时间：

```yaml
gateway.expected_nodes: 10
gateway.recover_after_time: 5m
```

这意味着 Elasticsearch 会采取如下操作：

- 等待集群至少存在 8 个节点
- 等待 5 分钟，或者10 个节点上线后，才进行数据恢复，这取决于哪个条件先达到。

这三个设置可以在集群重启的时候避免过多的分片交换。这可能会让数据恢复从数个小时缩短为几秒钟。

注意：这些配置只能设置在 `config/elasticsearch.yml` 文件中或者是在命令行里（它们不能动态更新）它们只在整个集群重启的时候有实质性作用。

### 5）最好使用单播代替组播

Elasticsearch 默认被配置为使用单播发现，以防止节点无意中加入集群。只有在同一台机器上运行的节点才会自动组成集群。

虽然组播仍然作为插件提供， 但它**应该永远不在生产环境使用**了，否在你得到的结果就是一个节点意外的加入到了你的生产环境，仅仅是因为他们收到了一个错误的组播信号。 对于组播 *本身* 并没有错，组播会导致一些愚蠢的问题，并且导致集群变的脆弱（比如，一个网络工程师正在捣鼓网络，而没有告诉你，你会发现所有的节点突然发现不了对方了）。

使用单播，你可以为 Elasticsearch 提供一些它应该去尝试连接的节点列表。 当一个节点联系到单播列表中的成员时，它就会得到整个集群所有节点的状态，然后它会联系 master 节点，并加入集群。

这意味着你的单播列表不需要包含你的集群中的所有节点， 它只是需要足够的节点，当一个新节点联系上其中一个并且说上话就可以了。如果你使用 master 候选节点作为单播列表，你只要列出三个就可以了。 这个配置在 `elasticsearch.yml` 文件中：

```yaml
discovery.zen.ping.unicast.hosts: ["host1", "host2:port"]
```

## 3. 不要触碰这些配置！

### 1）垃圾回收器

Elasticsearch 默认的垃圾回收器（ GC ）是 CMS。 这个垃圾回收器可以和应用并行处理，以便它可以最小化停顿。 然而，它有两个 stop-the-world 阶段，处理大内存也有点吃力。

尽管有这些缺点，它还是目前对于像 Elasticsearch 这样低延迟需求软件的最佳垃圾回收器。官方建议使用 CMS。

G1GC 还是太新了，经常发现新的 bugs。这些错误通常是段（ segfault ）类型，便造成硬盘的崩溃。 Lucene 的测试套件对垃圾回收算法要求严格，看起来这些缺陷 G1GC 并没有很好地解决。

### 2）线程池

Elasticsearch 默认的线程设置已经是很合理的了。对于所有的线程池（除了 `搜索` ），线程个数是根据 CPU 核心数设置的。 如果你有 8 个核，你可以同时运行的只有 8 个线程，只分配 8 个线程给任何特定的线程池是有道理的。

搜索线程池设置的大一点，配置为 `int（（ 核心数 ＊ 3 ）／ 2 ）＋ 1` 。

## 4. 文件描述符和 MMap

Lucene 使用了 *大量的* 文件。 同时，Elasticsearch 在节点和 HTTP 客户端之间进行通信也使用了大量的套接字（注：sockets）。 所有这一切都需要足够的文件描述符。

可悲的是，许多现代的 Linux 发行版本，每个进程默认允许一个微不足道的 1024 文件描述符。这对一个小的 Elasticsearch 节点来说实在是太 *低* 了，更不用说一个处理数以百计索引的节点。

**你应该增加你的文件描述符，设置一个很大的值**，如 64,000。参考[这里](https://www.tecmint.com/increase-set-open-file-limits-in-linux/)。

```js
GET /_nodes/process

{
   "cluster_name": "elasticsearch__zach",
   "nodes": {
      "TGn9iO2_QQKb0kavcLbnDw": {
         "name": "Zach",
         "transport_address": "inet[/192.168.1.131:9300]",
         "host": "zacharys-air",
         "ip": "192.168.1.131",
         "version": "2.0.0-SNAPSHOT",
         "build": "612f461",
         "http_address": "inet[/192.168.1.131:9200]",
         "process": {
            "refresh_interval_in_millis": 1000,
            "id": 19808,
            "max_file_descriptors": 64000, 
            "mlockall": true
         }
      }
   }
}
```

`max_file_descriptors` 字段显示 Elasticsearch 进程可以访问的可用文件描述符数量。

Elasticsearch 对各种文件混合使用了 NioFs（ 注：非阻塞文件系统）和 MMapFs （ 注：内存映射文件系统）。请确保你配置的最大映射数量，以便有足够的虚拟内存可用于 mmapped 文件。这可以暂时设置：

```js
sysctl -w vm.max_map_count=262144
```

或者你可以在 `/etc/sysctl.conf` 通过修改 `vm.max_map_count` 永久设置它。

# 生产部署后

## 1. 动态变更设置

Elasticsearch 里很多设置都是动态的，可以通过 API 修改。

`集群更新` API 有两种工作模式：

- 临时（Transient）

  这些变更在集群重启之前一直会生效。一旦整个集群重启，这些配置就被清除。

- 永久（Persistent）

  这些变更会永久存在直到被显式修改。即使全集群重启它们也会存活下来并覆盖掉静态配置文件里的选项。

```js
PUT /_cluster/settings
{
    "persistent" : {
        "discovery.zen.minimum_master_nodes" : 2 
    },
    "transient" : {
        "indices.store.throttle.max_bytes_per_sec" : "50mb" 
    }
}
```

可以动态更新的设置的完整清单，请阅读 [online reference docs](https://www.elastic.co/guide/en/elasticsearch/reference/master/cluster-update-settings.html)。

## 2. 日志记录

Elasticsearch 会输出很多日志，都放在 `ES_HOME/logs` 目录下。默认的日志记录等级是 `INFO` 。 它提供了适度的信息，但是又设计好了不至于让你的日志太过庞大。

让我们调高节点发现的日志记录级别：

```js
PUT /_cluster/settings
{
    "transient" : {
        "logger.discovery" : "DEBUG"
    }
}
```

设置失效，Elasticsearch 将开始输出 `discovery` 模块的 `DEBUG` 级别的日志。

### 1）慢日志

这个日志的目的是捕获那些超过指定时间阈值的查询和索引请求。这个日志用来追踪由用户产生的很慢的请求很有用。

默认情况，慢日志是不开启的。要开启它，需要定义具体动作（query，fetch 还是 index），你期望的事件记录等级（ `WARN` 、 `DEBUG` 等），以及时间阈值。

这是一个索引级别的设置，也就是说可以独立应用给单个索引：

```js
PUT /my_index/_settings
{
    // 查询慢于 10 秒输出一个 WARN 日志
    "index.search.slowlog.threshold.query.warn" : "10s",
    // 获取慢于 500 毫秒输出一个 DEBUG 日志
    "index.search.slowlog.threshold.fetch.debug": "500ms", 
    // 索引慢于 5 秒输出一个 INFO 日志
    "index.indexing.slowlog.threshold.index.info": "5s" 
}
```

你也可以在 `elasticsearch.yml` 文件里定义这些阈值。没有阈值设置的索引会自动继承在静态配置文件里配置的参数。一旦阈值设置过了，你可以和其他日志器一样切换日志记录等级：

```js
PUT /_cluster/settings
{
    "transient" : {
        "logger.index.search.slowlog" : "DEBUG", 
        "logger.index.indexing.slowlog" : "WARN" 
    }
}
```

## 3. 索引性能技巧

### 1）使用批量请求并调整其大小

批量的大小则取决于你的数据、分析和集群配置，不过每次批量数据 5–15 MB 大是个不错的起始点。

从 5–15 MB 开始测试批量请求大小，缓慢增加这个数字，直到你看不到性能提升为止。然后开始增加你的批量写入的并发度（多线程等等办法）。

用 Marvel 以及诸如 `iostat` 、 `top` 和 `ps` 等工具监控你的节点，观察资源什么时候达到瓶颈。如果你开始收到 `EsRejectedExecutionException` ，你的集群没办法再继续了：至少有一种资源到瓶颈了。或者减少并发数，或者提供更多的受限资源（比如从机械磁盘换成 SSD），或者添加更多节点。

### 2）存储

这里有一些优化磁盘 I/O 的技巧：

- 使用 SSD。
- 使用 RAID 0。条带化 RAID 会提高磁盘 I/O，代价显然就是当一块硬盘故障时整个就故障了。不要使用镜像或者奇偶校验 RAID 因为副本已经提供了这个功能。
- 另外，使用多块硬盘，并允许 Elasticsearch 通过多个 `path.data` 目录配置把数据条带化分配到它们上面。
- 不要使用远程挂载的存储，比如 NFS 或者 SMB/CIFS。这个引入的延迟对性能来说完全是背道而驰的。
- 如果你用的是 EC2，当心 EBS。即便是基于 SSD 的 EBS，通常也比本地实例的存储要慢。

### 3）段和合并

Elasticsearch 会自动对大量段合并进行限流操作，限流阈值默认值是 20 MB/s，对机械磁盘应该是个不错的设置。如果你用的是 SSD，可以考虑提高到 100–200 MB/s。测试验证对你的系统哪个值合适：

```js
PUT /_cluster/settings
{
    "persistent" : {
        "indices.store.throttle.max_bytes_per_sec" : "100mb"
    }
}
```

如果你在做批量导入，完全不在意搜索，你可以彻底关掉合并限流。这样让你的索引速度跑到你磁盘允许的极限：

```js
PUT /_cluster/settings
{
    "transient" : {
        "indices.store.throttle.type" : "none" 
    }
}
```

设置限流类型为 `none` 彻底关闭合并限流。等你完成了导入，记得改回 `merge` 重新打开限流。

如果你**使用的是机械磁盘而非 SSD**，你需要添加下面这个配置到你的 `elasticsearch.yml` 里：

```yaml
index.merge.scheduler.max_thread_count: 1
```

机械磁盘在并发 I/O 支持方面比较差，所以我们需要降低每个索引并发访问磁盘的线程数。这个设置允许 `max_thread_count + 2` 个线程同时进行磁盘操作，也就是设置为 `1` 允许三个线程。

最后，你可以增加 `index.translog.flush_threshold_size` 设置，从默认的 512 MB 到更大一些的值，比如 1 GB。这可以在一次清空触发的时候在事务日志里积累出更大的段。而通过构建更大的段，清空的频率变低，大段合并的频率也变低。这一切合起来导致更少的磁盘 I/O 开销和更好的索引速率。当然，**你会需要对应量级的 heap 内存用以积累更大的缓冲空间**，调整这个设置的时候请记住这点。

### 4）其他

最后，还有一些其他值得考虑的东西需要记住：

- 如果你的搜索结果不需要近实时的准确度，考虑把每个索引的 `index.refresh_interval` 改到 `30s`。如果你是在做**大批量导入**，导入期间你可以通过设置这个值为 `-1` 关掉刷新。别忘记在完工的时候重新开启它。

- 如果你在做**大批量导入**，考虑通过设置 `index.number_of_replicas: 0`关闭副本。文档在复制的时候，整个文档内容都被发往副本节点，然后逐字的把索引过程重复一遍。这意味着每个副本也会执行分析、索引以及可能的合并过程。

  相反，如果你的索引是零副本，然后在写入完成后再开启副本，恢复过程本质上只是一个字节到字节的网络传输。相比重复索引过程，这个算是相当高效的了。

- 如果你没有给每个文档自带 ID，使用 Elasticsearch 的自动 ID 功能。 这个为避免版本查找做了优化，因为自动生成的 ID 是唯一的。

- 如果你在使用自己的 ID，尝试使用一种 [Lucene 友好的](http://blog.mikemccandless.com/2014/05/choosing-fast-unique-identifier-uuid.html) ID。包括零填充序列 ID、UUID-1 和纳秒；这些 ID 都是有一致的，压缩良好的序列模式。相反的，像 UUID-4 这样的 ID，本质上是随机的，压缩比很低，会明显拖慢 Lucene。

## 4. 推迟分片分配

Elasticsearch 将自动在可用节点间进行分片均衡，包括新节点的加入和现有节点的离线。

理论上来说，这个是理想的行为，我们想要提拔副本分片来尽快恢复丢失的主分片。 我们同时也希望保证资源在整个集群的均衡，用以避免热点。

为了解决这种瞬时中断的问题，Elasticsearch 可以推迟分片的分配。这可以让你的集群在重新分配之前有时间去检测这个节点是否会再次重新加入。

### 1）修改默认延时

通过修改参数 `delayed_timeout` ，默认等待时间可以全局设置也可以在索引级别进行修改:

```js
PUT /_all/_settings 
{
  "settings": {
    "index.unassigned.node_left.delayed_timeout": "5m" 
  }
}
```

通过使用 `_all` 索引名，我们可以为集群里面的所有的索引使用这个参数。默认时间被修改成了 5 分钟。

这个配置是动态的，可以在运行时进行修改。如果你希望分片立即分配而不想等待，你可以设置参数： `delayed_timeout: 0`.

### 2）自动取消分片迁移

如果节点在超时之后再回来，且集群还没有完成分片的移动，会发生什么事情呢？在这种情形下， Elasticsearch 会检查该机器磁盘上的分片数据和当前集群中的活跃主分片的数据是不是一样 — 如果两者匹配， 说明没有进来新的文档，包括删除和修改 — 那么 master 将会取消正在进行的再平衡并恢复该机器磁盘上的数据。

之所以这样做是因为本地磁盘的恢复永远要比网络间传输要快，并且我们保证了他们的分片数据是一样的，这个过程可以说是双赢。

如果分片已经产生了分歧（比如：节点离线之后又索引了新的文档），那么恢复进程会继续按照正常流程进行。重新加入的节点会删除本地的、过时的数据，然后重新获取一份新的。

## 5. 滚动重启

总有一天你会需要做一次集群的滚动重启——保持集群在线和可操作，但是逐一把节点下线。

常见的原因：Elasticsearch 版本升级，或者服务器自身的一些维护操作（比如操作系统升级或者硬件相关）。不管哪种情况，都要有一种特别的方法来完成一次滚动重启。

我们需要的是，告诉 Elasticsearch 推迟再平衡，因为对外部因子影响下的集群状态，我们自己更了解。操作流程如下：

1. 可能的话，停止索引新的数据。虽然不是每次都能真的做到，但是这一步可以帮助提高恢复速度。

2. 禁止分片分配。这一步阻止 Elasticsearch 再平衡缺失的分片，直到你告诉它可以进行了。如果你知道维护窗口会很短，这个主意棒极了。你可以像下面这样禁止分配：
    ```js
    PUT /_cluster/settings
    {
        "transient" : {
            "cluster.routing.allocation.enable" : "none"
        }
    }
    ```

3. 关闭单个节点。

4. 执行维护/升级。

5. 重启节点，然后确认它加入到集群了。

6. 用如下命令重启分片分配：

   ```js
   PUT /_cluster/settings
   {
       "transient" : {
           "cluster.routing.allocation.enable" : "all"
       }
   }
   ```

   分片再平衡会花一些时间。一直等到集群变成 `绿色` 状态后再继续。 

7. 重复第 2 到 6 步操作剩余节点。

8. 到这步你可以安全的恢复索引了（如果你之前停止了的话），不过等待集群完全均衡后再恢复索引，也会有助于提高处理速度。

## 6. 备份与恢复

要备份你的集群，你可以使用 `snapshot` API。

要使用这个功能，你必须首先创建一个保存数据的仓库。有多个仓库类型可以供你选择：

- 共享文件系统，比如 NAS
- Amazon S3
- HDFS (Hadoop 分布式文件系统)
- Azure Cloud

详情参考[这里](https://elasticsearch.cn/book/elasticsearch_definitive_guide_2.x/backing-up-your-cluster.html) 。

# 参考

1. [Elasticsearch: 权威指南 » 管理、监控和部署](https://elasticsearch.cn/book/elasticsearch_definitive_guide_2.x/administration.html)
