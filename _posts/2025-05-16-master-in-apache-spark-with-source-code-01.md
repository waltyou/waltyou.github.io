---
layout: post
title: 精通 Apache Spark 源码 | 问题 01・一、核心架构 | SparkContext 初始化链路（DAGScheduler/TaskScheduler 与集群管理器交互）
date: 2025-05-16 13:39:04
author: admin
comments: true
categories: [Spark]
tags: [BigData, Spark]
mermaid: true
---

<!-- more -->

---

* 目录
{:toc}
---

## 前言

这是[本系列](../master-in-apache-spark-with-source-code-00)的第一个问题，属于第一大类“核心架构”，问题描述为：

```markdown
**Trace SparkContext initialization**: What key components (DAGScheduler, TaskScheduler, SchedulerBackend) are created, and how do they interact with cluster managers (YARN/Kubernetes/Standalone)?  
   *Key files:* `SparkContext.scala`, `SparkSession.scala`
```

## 解答

### 1. 入口 SparkSession.scala

`SparkSession` 从 Spark 2.0 开始是 Spark SQL 的主要编程入口，路径: `sql/core/src/main/scala/org/apache/spark/sql/SparkSession.scala`，它的主要功能是作为 Spark SQL 的入口点,用于创建和管理 DataFrame/Dataset API。

它有三个部分：

1. Class SparkSession
2. Object SparkSession
3. Class Builder

> Scala 中的class不持有静态方法或者变量，统一都在同名的object下面

#### 1.1 Class SparkSession

它是一个私有构造的类，实现了 Serializable 和 Closeable 接口， 每个 SparkSession 包含:

* SparkContext
* SessionState (会话状态)
* SharedState (共享状态)
* RuntimeConfig (运行时配置)

数据操作

```scala
createDataFrame()    // 从各种数据源创建
read                 // 读取数据
readStream           // 读取流数据
sql()                // 执行 SQL 查询
```

目录服务

```scala
catalog              // 管理数据库、表、函数等
table()              // 访问表/视图
```

#### 1.2 Object SparkSession 

主要是builder class以及一些操作session的静态方法，如：

```scala
// 静态方法
setActiveSession()        // 设置当前活跃会话
clearActiveSession()      // 清除活跃会话
setDefaultSession         // 设置默认会话
...
```

#### 1.3 Class Builder 

用于构建 SparkSession 的构建器，主要配置方法:

```scala
.master()        // 设置运行模式
.appName()       // 设置应用名称
.config()        // 设置配置项
.enableHiveSupport() // 启用 Hive 支持
.withExtensions()  //注入扩展
```

#### 1.4 其他特性

单例模式

* 使用 ThreadLocal 确保线程安全： `private val activeThreadSession = new InheritableThreadLocal[SparkSession]`
* 维护全局默认会话
* 支持每个线程独立的活跃会话


扩展机制

* 支持通过 SparkSessionExtensions 进行扩展
* 可以添加:
  * 分析器规则
  * 优化器规则
  * 规划策略
  * 自定义解析器


这个拓展机制还挺有意思，之前都是通过其它方式注入这些规则的，并没有一个统一的入口。


### 2. 仔细看看 builder.getOrCreate() 方法

SparkSession 的实例是方法 builder.getOrCreate() 创建的，简约的过程如下：

```scala
def getOrCreate(): SparkSession = synchronized {
  // 1. 创建 SparkConf
  val sparkConf = new SparkConf()
  
  // 2. 创建或获取 SparkContext
  val sparkContext = userSuppliedContext.getOrElse {
    // 创建新的 SparkContext
    SparkContext.getOrCreate(sparkConf)
  }
  
  // 3. 创建 SparkSession
  session = new SparkSession(sparkContext, ...)
}
```

### 3. SparkContext 创建过程

函数 `SparkContext.getOrCreate(sparkConf)`，就是在保证线程安全的情况下，new了一个新的 SparkContext，所以直接来看SparkContext 的构造函数，简单起见，只用代码标出主要流程，和实际代码可能会不同。

```scala
// 1. 构造函数入口
class SparkContext(config: SparkConf) {
  // 2. 记录创建现场
  private val creationSite = Utils.getCallSite()
  
  // 3. 标记正在构造
  SparkContext.markPartiallyConstructed(this)

  // 4. 初始化核心组件
  _conf = config.clone()  // 配置
  _env = createSparkEnv() // SparkEnv
  _statusTracker = new SparkStatusTracker(this) // 状态跟踪
  _progressBar = new ConsoleProgressBar() // 进度条
  _ui = new SparkUI() // UI界面
  
  // 5. 创建和启动调度系统
  val (sched, ts) = SparkContext.createTaskScheduler(this, master)
  _schedulerBackend = sched
  _taskScheduler = ts
  _dagScheduler = new DAGScheduler(this)
  
  // 6. 最后标记为激活状态
  SparkContext.setActiveContext(this)
}
```

可以看出来它初始化了很多组件，比如 SparkEnv， statusTracker， SchedulerBackend， TaskScheduler，DAGScheduler, SparkUI 等。来看一看问题涉及到的几个关键部件。

#### 3.1 SchedulerBackend & TaskScheduler

这两个组件都是通过函数 createTaskScheduler 创建的，它根据给定的 master URL 创建任务调度器 TaskScheduler。

核心功能：

* 根据不同的 master URL 格式创建对应的调度器组件
* 配置本地运行或集群运行模式下的任务调度
* 处理任务失败的重试机制

主要参数：

* sc: SparkContext - Spark 上下文实例
* master: String - master URL,定义了 Spark 的运行模式

对 master 这个字符串进行模式匹配：

* "local" - 使用1个线程在本地运行
* "local[N]" - 使用N个线程在本地运行
* "local[*]" - 使用所有可用CPU线程在本地运行 
* "local[N,M]" - 使用N个线程,每个任务最多失败M次
* "local-cluster" - 本地集群模式
* "spark://host:port" - 连接到 Spark standalone cluster
* others：
  * "yarn" - 连接到 YARN 集群
  * "mesos://host:port" - 连接到 Mesos 集群
  * "k8s://host:port" - 连接到 Kubernetes 集群

这里值得一提的是这些others的cluster manager，他们都是先通过`getClusterManager`方法一个 ExternalClusterManager 的实现，然后调用它的 createTaskScheduler createSchedulerBackend 方法初始化taskScheduler和schedulerBackend。各自的实现都在不同的moduler下面，如：

* Yarn: resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClusterManager.scala
* Mesos: resource-managers/mesos/src/main/scala/org/apache/spark/scheduler/cluster/mesos/MesosClusterManager.scala
* k8s: resource-managers/kubernetes/core/src/main/scala/org/apache/spark/scheduler/cluster/k8s/KubernetesClusterManager.scala

这种模块化设计有以下好处:

* 保持核心代码的简洁性
* 允许按需加载不同的集群管理器
* 避免不必要的依赖关系
* 便于维护和扩展

虽然在核心的 SparkContext 中看不到这些具体实现,但它们是通过插件式架构在运行时动态加载的。

来看看master=local时:

```scala
checkResourcesPerTask(1)
val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
val backend = new LocalSchedulerBackend(sc.getConf, scheduler, 1)
scheduler.initialize(backend)
```

`TaskSchedulerImpl` 是一个核心调度器实现类，负责在不同类型的集群（如本地、本地集群、YARN、K8s等）上调度和管理任务。它通过与 `SchedulerBackend` 交互，实现任务的分发、资源分配、失败重试、推测执行等功能。它首先调用initialize()和start()，然后通过submitTasks方法提交任务集。主要方法：

* submitTasks：提交新的 TaskSet。
* resourceOffers：接收资源 offer 并分配任务。
* statusUpdate：处理任务状态更新。
* executorLost、removeExecutor：处理 executor 丢失。
* checkSpeculatableTasks：检查是否有任务需要推测执行。
* stop：停止调度器，清理资源。

> TaskSetManager 是单个 Stage 内任务生命周期的管理者和调度者。

submitTasks 会构建一个 taskSetManager，然后调用 schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)，schedulableBuilder里面有个 rootPool 变量，它会存有所有的tasks。

接着当 resourceOffers 接收集群管理器（如 YARN、K8s）提供的资源 offer后，它会做如下事情：先过滤、整理资源，获取具体任务（val sortedTaskSets = rootPool.getSortedTaskSetQueue），再按调度策略和本地性分配任务，每个任务分配前会检查资源是否满足需求，支持 barrier、推测执行、健康检查等高级调度特性，任务分配后会更新内部状态，确保资源和任务的正确映射。


`LocalSchedulerBackend` 类比较简单，它用于在单 JVM 内调度和管理任务，实现了 SchedulerBackend 和 ExecutorBackend 接口，主要功能有这些：

1. 启动本地调度后端：初始化本地端点、连接 Launcher、注册 Executor，并设置应用状态为 RUNNING
2. 停止调度后端：关闭本地端点，设置应用状态为 FINISHED 或 KILLED。
3. 资源调度与任务分配：通过本地端点 localEndpoint 发送 ReviveOffers 消息，触发资源调度。

主要要注意class LocalEndpoint 中的 reviveOffers 函数，它调用 scheduler.resourceOffers 获取 taskDescription，然后交给executor 去launchTask：

```scala
def reviveOffers(): Unit = {
  // local mode doesn't support extra resources like GPUs right now
  val offers = IndexedSeq(new WorkerOffer(localExecutorId, localExecutorHostname, freeCores,
    Some(rpcEnv.address.hostPort)))
  for (task <- scheduler.resourceOffers(offers, true).flatten) {
    freeCores -= scheduler.CPUS_PER_TASK
    executor.launchTask(executorBackend, task)
  }
}
```

再来看看 master=yarn时， YarnClusterManager 有client， cluster 两种模式。

taskScheduler 在client模式下是 YarnScheduler，它直接继承了 TaskSchedulerImpl，只是重写了func getRacksForHosts，通过hadoopConfiguration获取对应主机的机架信息。SchedulerBackend 是 YarnClientSchedulerBackend。在cluster模式下是 YarnClusterScheduler，继承了 YarnScheduler，重写了方法postStartHook，在里面调用了 `ApplicationMaster.sparkContextInitialized(sc)`.

schedulerBackend 最好是整体角度学习，因为它复用了很多CoarseGrainedSchedulerBackend（通用实现）的方法。整体依赖关系如下：SchedulerBackend（接口）→ CoarseGrainedSchedulerBackend（通用实现）→ StandaloneSchedulerBackend/YarnSchedulerBackend（具体集群实现）→ YarnClientSchedulerBackend（YARN client模式实现）/ YarnClusterSchedulerBackend（YARN cluster模式实现）。

依次介绍一下各个的主要功能。

SchedulerBackend（trait）

* Spark调度后端的通用接口，定义了调度器与集群管理器交互的基本方法。
* 包括启动、停止、资源请求、任务杀死、应用ID获取等接口。

CoarseGrainedSchedulerBackend

* Spark调度器后端，负责管理Executor的注册、移除、资源请求、任务分发等。
* 维护Executor的状态、资源信息、注册情况等。
* 通过DriverEndpoint与Executor通信，处理各种调度和资源管理消息。
* 支持Executor的动态申请、移除、去除、Decommission等操作。
* 提供与集群管理器（如Standalone、Yarn等）的抽象接口。

StandaloneSchedulerBackend

* 继承自CoarseGrainedSchedulerBackend，实现Spark Standalone模式下的调度后端。
* 负责与Standalone Master/Worker通信，管理Executor的生命周期。
* 实现Executor的注册、移除、去除、Decommission等具体逻辑。
* 处理Standalone集群特有的资源管理和Executor丢失检测（快慢路径）。

YarnSchedulerBackend

* 继承自CoarseGrainedSchedulerBackend，实现YARN模式下的调度后端。
* 负责与YARN ApplicationMaster通信，管理Executor的分配和回收。
* 处理YARN特有的资源请求、Executor丢失原因查询、AM注册等。
* 支持YARN的多次尝试（ApplicationAttemptId）、WebUI代理等功能。

YarnClientSchedulerBackend

* 继承自YarnSchedulerBackend，实现YARN client模式下的调度后端。
* 负责通过YARN Client提交应用，监控应用状态，处理异常终止等。
* 维护与YARN ResourceManager的连接，处理应用的启动、监控和关闭。


`YarnClientSchedulerBackend` 和 `YarnClusterSchedulerBackend` 都继承自 `YarnSchedulerBackend`，但它们分别用于 YARN 的 client 模式和 cluster 模式。源码层面的主要区别如下：

1. **应用提交方式不同**  
   - `YarnClientSchedulerBackend` 负责在 client 模式下通过 `Client` 类主动向 YARN ResourceManager 提交应用（`client.submitApplication()`），并在本地 driver 进程中运行 driver 逻辑。
   - `YarnClusterSchedulerBackend` 用于 cluster 模式，driver 运行在 YARN ApplicationMaster 容器中，通过 `ApplicationMaster.getAttemptId` 获取应用信息并绑定。

2. **启动流程不同**  
   - `YarnClientSchedulerBackend.start()`：先调用 `super.start()`，然后创建 `Client`，提交应用，等待应用运行，并启动监控线程监控应用状态。
   - `YarnClusterSchedulerBackend.start()`：获取 ApplicationMaster 的 attemptId，绑定到 YARN，调用 `super.start()`，不需要本地提交应用。

3. **监控与退出机制不同**  
   - `YarnClientSchedulerBackend` 有专门的 `MonitorThread`，监控 YARN 应用状态，发现异常时会主动调用 `sc.stop()` 并可能退出 JVM。
   - `YarnClusterSchedulerBackend` 没有类似的本地监控线程，driver 生命周期由 YARN ApplicationMaster 管理。

4. **日志和属性获取方式不同**  
   - `YarnClusterSchedulerBackend` 重写了 `getDriverLogUrls` 和 `getDriverAttributes`，通过 YARN 容器信息获取 driver 日志和属性。
   - `YarnClientSchedulerBackend` 没有重写这些方法。



#### 3.2 DAGScheduler

To be continue....