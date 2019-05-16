---
layout: post
title: Hadoop Yarn的架构
date: 2018-05-17 14:16:04
author: admin
comments: true
categories: [Hadoop]
tags: [Hadoop, Big Data, Yarn]
---

Yarn是Hadoop 2.x版本后，抽象出来的新的资源管理层，它关注的事情更加集中：资源管理。

<!-- more -->
---



* 目录
{:toc}
---

# 一、概述

Yarn（Yet Another Resource Negotiator），是Hadoop的资源管理层，它旨在提供一个通用且灵活的框架来管理Hadoop集群中的计算资源。

在它之上可以运行多种计算引擎，并且用这些引擎来处理HDFS中存储的数据。

[![](/images/posts/hadoop-yarn-architecture-tutorial.png)](/images/posts/hadoop-yarn-architecture-tutorial.png)


---

# 二、架构总览

Yarn也是master/slave结构，同时它有主要有3个部件：

1. Resource Manager (RM)
2. Node Manager (NM)
3. Application Master (AM)

[![](/images/posts/Apache-YARN-architecture-min.jpg)](/images/posts/Apache-YARN-architecture-min.jpg)

它们分别都是做什么呢？接下来详细介绍。

---

# 三、Resource Manager (RM)

## 1. 简介

它是Yarn的master守护进程。可以类比与MRV1中的JobTracker。

它管理着整个集群所有的计算资源和内存资源，它也负责仲裁两个application之间的资源竞争。

### 1）两个主要组件

1. Scheduler

    Scheduler调度器，负责为正在运行的application分配资源。

    scheduler是个纯粹的调度程序，这意味着它不执行任何监视，也不会执行对application的跟踪，甚至不能保证重新启动那些由于application故障或硬件故障而失败的任务。

    它基于每个application的资源需求，来执行它的调度函数。这个资源可以包括CPU，内存，网络，硬盘等。

    这个分配原则会依据可用资源的情况以及配置的共享策略。

    Scheduler程序有一个可插入的策略插件，它负责在各种队列，应用程序等之间分配群集资源。可以参考现在的Map-Reduce调度程序（如CapacityScheduler和FairScheduler），是插件的一些示例。

2. Application manager

    ApplicationsManager负责维护提交的应用程序的集合，也就是说它管理着集群中那些运行的 Application Master（AM）。

    它接受来自客户端的作业并协商容器以执行特定于应用程序的ApplicationMaster，并在出现故障时提供重新启动ApplicationMaster的服务。

    它同时维护了一个包含已完成application们的缓存，这样子在很长时间后，也可以通过从Web UI或者命令行来服务客户的各种请求。

## 2. 所有组件

先来看下总体架构：

[![](/images/posts/Resource-Manager.png)](/images/posts/Resource-Manager.png)

从上图可以看出，这些组件一起工作，完成了RM完整的功能。组件分为以下6大类，我们依次来看。

### 1）将RM连接到客户端的组件们

1. ClientService

    RM的client接口，这个组件处理了从客户端到RM的所有RPC接口，其中包括任务提交，任务终止，获取队列信息、集群状态等。

2. AdminService

    为了避免由于集群忙于处理普通用户的请求，而使admin的请求“被饿死”的这种情况出现，同时也为了给admin命令更高的优先级，
    所有admin的操作，如刷新节点、队列配置等，都将通过AdminService这个额外接口进行服务。

### 2）将RM连接到节点的组件们

1. ResourceTrackerService

    这个组件获取集群中节点的心跳信号，并把它们转发给YarnScheduler。
    响应来自所有节点的RPC请求，注册新的节点，拒绝任何来自无效、退役节点的请求，它和NMLivelinessMonitor、NodesListManager紧密协作。

2. NMLivelinessMonitor

    这个是为了追踪存活的节点和死掉的节点。它追踪着每个节点的最后一次心跳信号。
    如果一个节点没有在配置的时间阀值（默认10min）之内发送心跳，就会被当作死掉的节点。
    同时在这个节点上运行的container都会被标记为死亡，同时也不会安排新的container给这个节点。

3. NodesListManager

    管理有效可执行的节点。
    负责读取主机配置文件，并根据这些文件为节点初始列表分类。
    跟踪随着时间推移而退役的节点。

### 3）与每个应用程序AM进行交互的组件

1. ApplicationMasterService

    服务来自所有AM的RPC请求，比如：注册新的AM;终止或者注销任一完成的AM;从所有运行的AM获取容器分配和解除分配请求，并将这些请求转发给YarnScheduler.
    因此，ApplicationMasterService 和 AMLivelinessMonitor 协同工作以保持应用程序主控的容错性。
2. AMLivelinessMonitor

    维护活动AM和死/无响应AM的列表，其职责是跟踪活动AM，通常在心跳帮助下跟踪AM死或活，并从资源管理器注册和取消注册AM。
    因此，当前所有正在运行或已分配给过期AM的容器，都将被标记为死亡。

### 4）ResourceManager的核心 - scheduler调度程序和相关组件

1. ApplicationsManager

    负责维护一个已提交应用的集合。它同时维护了一个包含已完成application们的缓存，这样子在很长时间后，也可以通过从Web UI或者命令行来服务客户的各种请求。
2. ApplicationACLsManager

    RM需要将面向API的用户设置为只能由授权用户访问，例如客户端和管理员请求。
    这个组件对每个应用维护了ACL列表，并且无论任何时候收到一个请求（如杀掉应用、查看应用状态），都会强制执行授权检测。
3. ApplicationMasterLauncher

    维护了一个线程池用来启动AM，这些AM或者来自那些新提交的应用，或者来自之前因为某些原因导致AM attempt退出的应用。
    同时当应用正常完成时，或者被终止后，它负责清理那些AM，
4. YarnScheduler

    YarnScheduler负责分配资源给那些受容量、队列等限制的各个正在运行的应用们。
    同时它还根据应用们的资源请求，执行其调度函数。例如对CPU，内存、硬盘、网络等。
5. ContainerAllocationExpirer

    该组件负责确保所有分配的容器都被AM使用，并随后在相应的NM上启动。
    因为AM是作为不可信的用户代码运行，它可能在不使用资源的情况下保留分配，这就可能导致集群利用率不足。
    为解决这个问题，ContainerAllocationExpirer维护在相应的NM上仍未使用的分配容器的列表。
    对于任何容器，如果相应的NM没有在配置的时间间隔（默认情况下为10分钟）内，向RM报告该容器已开始运行，则该容器被认为已经死亡并且已经过期。

### 5）TokenSecretManagers (为了安全)

Yarn拥有一套SecretManagers，用于管理令牌的费用/责任，用于在各种RPC接口上验证/授权请求的密钥。

1. ApplicationTokenSecretManager

    RM使用称为ApplicationTokens的每个应用token，来避免任意进程都可以来发送RM调度请求。
    该组件将每个令牌保存在本地内存中，直到应用程序结束。
    然后使用它来验证来自有效的AM进程的任何请求。
2. ContainerTokenSecretManager

    RM向特定节点上的容器，颁发名为Container Tokens的特殊标记到AM上。
    因此，AM使用这些令牌创建与NodeManager的连接，这些NM上有运行作业的容器。
3. RMDelegationTokenSecretManager

    它是ResourceManager特定的授权令牌秘密管理器。
    它负责为客户端生成委托令牌，这些令牌也可以传递给希望能够与RM交谈的未经认证的进程。

### 6）DelegationTokenRenewer

在安全模式下，RM被[Kerberos](https://zh.wikipedia.org/wiki/Kerberos)认证。因此提供了代表应用程序更新文件系统令牌的服务。

该组件会更新提交的应用程序的令牌，只要应用程序运行并且令牌不能再续订。

## 3. RM重启

RM是管理资源和调度在YARN上运行的应用程序的中央机构。所以它可能是yarn的SPOF。

它有两种重新启动方式：
1. 非工作保留RM重新启动

    此重新启动增强了RM以在可插拔状态存储中保持应用程序/尝试状态。
    资源管理器将在重新启动时从状态存储区重新载入相同的信息并重新启动以前运行的应用程序。用户不需要重新提交应用程序。
    节点管理器和客户端在RM停机期间将继续轮询RM直到RM出现，当RM出现时，它将通过心跳向所有正在与之通话的NM和AM发送重新同步命令。
    NM将杀死其所有它管理的container并重新注册到RM。
2. 保留工作RM重启

    重点在于通过在重启时，结合来自NM的容器状态，和来自AM的容器请求，来重构RM的运行状态。
    与非工作保持RM重新启动的关键区别在于，已经运行的应用程序不会在主启动后停止，因此应用程序不会由于RM /主服务器停机而丢失其处理的数据。
    RM通过利用所有节点管理器发送的容器状态来恢复其运行状态。当NM与重新启动的RM重新同步时，NM不会杀死容器。它继续管理容器，并在重新注册时将容器状态发送到RM。

## 4. RM高可用

高可用功能以Active / Standby ResourceManager对的形式添加冗余，以消除此单点故障。

ResourceManager HA通过Active/Standby体系结构实现。
在任何时间点，主服务器中的一个处于Active状态，其他资源管理器处于Standby模式，当Active发生任何事情时，他们正在等待接管。
启用自动故障转移时，转换为Active的触发，会来自admin（通过CLI）或通过集成故障转移控制器。

### 1）手动转换和故障转移

如果未配置自动故障转移，则管理员必须手动将其中一个RM转换为活动状态。
从主动主站到另一个主站的故障切换，他们需要将active master传输到standby，并将Standby-RM传输到Active。因此，这项活动可以使用YARN完成。

### 2）手动转换和故障转移

在这种情况下，不需要任何手动干预。
主服务器可以选择嵌入基于ActiveStandbyElector的Zookeeper（协调引擎）来决定哪个资源管理器应该处于活动状态。
当活动失败时，另一个资源管理器将自动选择为活动状态。

请注意，不需要运行单独的zookeeper守护程序，因为资源管理器中嵌入的ActiveStandbyElector充当故障检测器和领导者选举器，而不是单独的ZKFC守护程序。

---

# 四、Node Manager (NM)

## 1.简介

它是Yarn的slave守护进程。NM负责监控容器的资源使用情况并向ResourceManager报告，管理该机器上的用户进程，也会跟踪其运行节点的运行状况。

该设计还允许将长期运行的辅助服务插入到NM中; 这些是特定于应用程序的服务，作为配置的一部分进行指定，并在启动期间由NM加载。Shuffle是NM为YARN上的MapReduce应用程序提供的典型辅助服务。

## 2. 所有组件

先来看下总体架构：

[![](/images/posts/Node-Manager.jpg)](/images/posts/Node-Manager.jpg)

### 1）NodeStatusUpdater

启动时，此组件向ResourceManager（RM）注册并向每个节点发送有关可用资源的信息。随后的NM-RM通信交换更新每个节点的容器状态，如节点上运行的容器已完成容器等。

此外，RM可能会通知NodeStatusUpdater终止正在运行的容器。

### 2）Container Manager

作为NodeManager的核心组件，肩负着管理每个节点上运行的容器及其子组件的责任。这里每个子组件都会执行一些功能子集，这些功能子集正是管理节点上运行的容器所需的。

- RPC server

    接受来自AM的请求以启动新的容器，或者停止正在运行的容器。它与ContainerTokenSecretManager关联来授权所有请求。
    在此节点运行的容器上执行的所有操作，都会写入审计日志，可以使用安全工具进行后期处理。
- ResourceLocalizationService

    负责安全下载和组织各种容器所需的文件资源。它尽最大努力在所有可用磁盘上分发文件。它还强制下载文件的访问控制限制，并对它们设置适当的使用限制。
- ContainersLauncher

    保持一个线程池以尽可能快地准备和启动容器。此外，当RM或AM发送此类请求时，清理容器的进程。
- AuxServices

    NM提供了一个通过配置辅助服务来扩展其功能的框架。这允许特定框架可能需要的每节点自定义服务，并且仍然从其他NM中对其进行沙箱处理。这些服务必须在NM启动之前进行配置。
    当应用程序的第一个容器在节点上启动时以及应用程序被认为完成时，会通知辅助服务。
- ContainersMonitor

    该组件在容器运行时开始观察其资源利用率。为了强化内存等资源的隔离和公平共享，RM为每个容器分配了一定数量的这种资源。
    ContainersMonitor持续监视每个容器的使用情况，如果容器超出其分配范围，则表示容器被杀死。这样做是为了防止任何失控的容器对在同一节点上运行的其他行为良好的容器造成不利影响。
- LogHandler

    可插拔组件，可选择将容器的日志保留在本地磁盘上，或将它们压缩在一起并上传到文件系统。

### 3）Container Executor

与底层操作系统进行交互，以安全地放置容器所需的文件和目录，随后以安全的方式启动和清理与容器相对应的进程。

### 4）NodeHealthChecker Service

通过定期运行配置脚本来检查节点运行状况的功能是NodeHealthCheckerService的应有责任。

它还通过每隔一段时间在磁盘上创建临时文件来专门监视磁盘的运行状况。

系统健康状况的任何变化都会通知NodeStatusUpdater，NodeStatusUpdater会将信息传递给RM。

### 5）Security

- ContainerTokenSecretManager

    检查容器的传入请求，以确保所有传入请求确实由ResourceManager正确授权。

### 6）WebServer

公开应用程序列表，在给定时间点节点上运行的容器信息，节点健康相关信息以及容器生成的日志。

---

# 五、Application Master (AM)

一个Application Master运行每个应用。它从RM协商资源并与NM一起工作。它管理应用程序的生命周期。

AM在联系相应的NM以启动应用程序的各项任务之前，从RM的scheduler程序获取容器。

---

# 六、参考链接
1. How Hadoop MapReduce Works – MapReduce Tutorial: <https://data-flair.training/blogs/hadoop-yarn-tutorial/>
2. Hadoop YARN Resource Manager – A Yarn Tutorial: <https://data-flair.training/blogs/hadoop-yarn-resource-manager//>