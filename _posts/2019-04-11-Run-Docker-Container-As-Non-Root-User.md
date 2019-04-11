---
layout: post
title: 以非root用户启动 Docker container
date: 2019-04-11 12:16:04
author: admin
comments: true
categories: [Docker]
tags: [Docker]
---

启动 docker container后，默认的登陆用户为 root，那么如何以其他用户进入docker container 中呢？

<!-- more -->
---

## 目录
{:.no_toc}

* 目录
{:toc}
## 问题

启动 docker container后，默认的登陆用户为 root，那么你在容器中，创建一个目录或者文件时，它们的 owner 和 group 就都会是 root。如下：

```shell
docker run --rm -it \
  -v /local/path:/docker/path \ 
  --workdir /docker/path \  
  your-docker-image 
  mkdir testDir 
```

在这种情况下，如果你想在容器外，访问或者修改这些文件或目录的时候，就不太方便，因为你此时的用户可能是非root用户。

```shell
$ ll /local/path/
drwxr-x--- 2 root root        4096 Apr 11 11:22 testDir
```

## 解决方案

那么怎么办呢？

其实 docker 启动 container 时，可以加上一个 `--user` 的参数就可以了。这个参数接收两个id，一个 user id （UID），一个 group id （GID），由于docker 是和 系统共用同一个 kernel，所以它们之间共享 uid 、gid 列表。

如下：

```shell
docker run --rm -it \
  -v /local/path:/docker/path \ 
  --workdir /docker/path \  
  --user 1000:1000 \ 
  your-docker-image 
  mkdir testDir 
```

这样子打开 docker container 之后，在创建文件或者目录，它们的 owner 和 group 就都是 uid 为 1000、gid 也为1000的用户了。

如果不知道自己当前用户对应的 uid 和 gid 怎么办？可以用以下写法：

```shell
--user $(id -u):$(id -g)
```

它自动会取到当前用户的uid和gid。

## 参考资料

1. [Running a Docker container as a non-root user](https://medium.com/redbubble/running-a-docker-container-as-a-non-root-user-7d2e00f8ee15)

