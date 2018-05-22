---
layout: post
title: Docker一瞥
date: 2018-05-21 16:16:04
author: admin
comments: true
categories: [Docker]
tags: [Docker]
---

今天因为工作需要，拿到了一个dockerfile和一份代码，然后要让代码在docker里跑起来。然而自己之前没有接触docker，所以快速的学习了一下，这里做个总结。

<!-- more -->
---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 1. 前言

> Docker 是个划时代的开源项目，它彻底释放了计算虚拟化的威力，极大提高了应用的维护效率，降低了云计算应用开发的成本！使用 Docker，可以让应用的部署、测试和分发都变得前所未有的高效和轻松！

---

# 2. 什么是Docker

简单说就是个容器，它可以对进程进行封装隔离，这算是属于“操作系统层面的虚拟化技术”。

比起虚拟机要轻便许多，因为它直接借助宿主的内核，在上面运行进程。这样子它既不需要自己的内核，也不需要虚拟化硬件。

---

# 3. 有什么好处

1. 更高效的利用系统资源
2. 更快速的启动时间
3. 一致的运行环境
4. 持续交付和部署
5. 更轻松的迁移
6. 更轻松的维护和扩展

---

# 4. 关于docker的基本概念

## 4.1 镜像 Image

对于 Linux 而言，内核启动后，会挂载 root 文件系统为其提供用户空间支持。而 Docker 镜像（Image），就相当于是一个 root 文件系统。

## 4.2 容器 Container

镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。

## 4.3 仓库

镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。

一个 Docker Registry 中可以包含多个**仓库**（Repository）；每个仓库可以包含多个标签（Tag）；每个标签对应一个镜像。

---

# 5. 安装

我自己的电脑是Ubuntu 16.04，所以详细介绍这个，其他的可以到[官方安装指南](https://docs.docker.com/install/)了解。

1. 添加使用 HTTPS 传输的软件包以及 CA 证书

    ```shell
    $ sudo apt-get update

    $ sudo apt-get install \
        apt-transport-https \
        ca-certificates \
        curl \
        software-properties-common
    ```
2. 添加软件源的 GPG 密钥

    ```shell
    $ curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
    ```
3. 向 source.list 中添加 Docker 软件源(注意这是国内源)

    ```shell
    $ sudo add-apt-repository \
        "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu \
        $(lsb_release -cs) \
        stable"
    ```
4. 更新 apt 软件包缓存，并安装 docker-ce

    ```shell
    $ sudo apt-get update
    $ sudo apt-get install docker-ce
    ```

---

# 6. 怎么用

主要分两步：构造镜像，使用镜像。

## 6.1 使用 Dockerfile 定制镜像

重点来了，我手里现在有个dockerfile，怎么使用它呢？

先看一看dockerfile的内容：
```
FROM ubuntu:16.04

# Install common dependences
RUN apt-get update && \
    apt-get install -y \
    build-essential \
    git \
    wget \
    vim \
    apt-utils

# Install tesseract dependences
RUN apt-get install  -y \
    automake \
    libtool \
    autoconf \
    autoconf-archive \
    pkg-config \
    libpng12-dev
```

可以看出里面有两个关键词，**FROM** 和 **RUN**。

这是什么意思呢？

### 6.1.1 FROM 指定基础镜像

所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。

而 FROM 就是指定基础镜像，因此一个 Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令。

### 6.1.2 RUN 执行命令

RUN 指令是用来执行命令行命令的。我们可以通过RUN命令来对基础镜像进行修改，比如装软件、配置环境变量等基础操作。


> 除了以上两个命令外，Dockerfile也有许多其他命令，详情参考[链接](https://yeasy.gitbooks.io/docker_practice/image/dockerfile/)。


### 6.1.3 构建镜像

首先cd到dockerfile所在目录，然后执行：

```shell
$ docker build -t ubuntu:v2 .
```
这样子，我们就使用dockerfile构建了一个名字为“ubuntu:v2”的镜像。

docker build 命令进行镜像构建的格式为：

```
docker build [选项] <上下文路径/URL/->
```

到此为止dockerfile的使命已经完成了。

## 6.2 使用镜像

上一步完成构建了镜像，接下来要使用这个镜像。

### 6.2.1 使用bash

首先我们试着启动镜像里面的 bash 并且进行交互式操作：

```shell
$ docker run -t -i ubuntu:v2 /bin/bash

root@5b0063dd73a2:/#

```
这样子就进入了镜像中，我们可以在里面输入命令，比如cat、ls等。

最后使用exit退出。

### 6.2.2 在docker中运行本地代码

很多时候，我们的代码都是在本地机器上写好的，但是我们又想在docker环境中运行。

当然我们可以把代码打包ADD进docker镜像中、再解压的这个过程，写入dockerfile中，每次我们改变代码就重新build一次镜像，然后在进入docker的bash中运行我们的代码。但是这个过程太过繁琐，不适合调试。

所以我们需要了解docker的另一个概念：**Docker的数据管理**。

Docker 数据管理的主要分为两种：数据卷和挂载主机目录。

我这次主要使用**挂载主机目录**。

使用起来也很简单，使用**mount**命令就好：
```shell
$ docker run
    --mount type=bind,source=/your/code/path/,target=/path/in/docker/
    -it ubuntu:v2

root@5b0063dd73a2:/# ls /path/in/docker/

```
执行ls命令后，就可以看到自己本机的所有文件都在docker中出现了。

这个时候docker和本地文件系统是共享这个文件夹的，也就是说它们的写操作是同步的。如果你只想让docker对文件有读的权限，只需在mount后加上**readonly**即可：
```shell
--mount type=bind,source=/your/code/path/,target=/path/in/docker/,readonly
```

到此为止，我们就很欢快的调试代码了。

---

# 7. 参考资料

1. [Docker — 从入门到实践](https://yeasy.gitbooks.io/docker_practice/)

