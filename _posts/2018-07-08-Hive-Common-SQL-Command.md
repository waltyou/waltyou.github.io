---
layout: post
title: Hive 常用 Sql 命令
date: 2018-07-08 10:16:04
author: admin
comments: true
categories: [Hive]
tags: [Big Data, Hive]
---

Hive 提供了一个交互式接口，来让用户通过 SQL 来操作数据。这里记录一下常用的Hive SQL 语句。

<!-- more -->

---



* 目录
{:toc}
---

# 构建测试环境

在 docker hub上搜索一下自己感兴趣的 docker 镜像，通过pull命令将它拉到本地。

```
docker pull teradatalabs/cdh5-hive

docker run -d --name hadoop-master -h hadoop-master teradatalabs/cdh5-hive

docker exec -it hadoop-master bash
```

然后在docker中运行 hive 即可。

# 行列转换

现在有一张成绩表，内容如下：

name | subject | score
---|---|---
aaa | culture | 90
aaa | math | 98
aaa | english | 94
aaa | bio | 80
bbb | culture | 91
bbb | math | 93
bbb | english | 94
bbb | bio | 91

## 1. 行变列

如果我们想得到一张这样的表：

name | culture | math | english | bio
---|---|---|---|---
aaa | 90 | 98 | 94 | 80
bbb | 91 | 93 | 94 | 91

怎么办呢？

```sql
create table row_scores as 
select 
name, 
MAX(CASE WHEN subject="culture" THEN score ELSE 0 END) as culture,
MAX(CASE WHEN subject="math" THEN score ELSE 0 END) as math,
MAX(CASE WHEN subject="english" THEN score ELSE 0 END) as english,
MAX(CASE WHEN subject="bio" THEN score ELSE 0 END) as bio
FROM scores
GROUP BY name;
```

## 2. 列变行

```sql
create table column_scores as 
select name, 'culture' as subject, culture as score from row_scores
union all
select name, 'math' as subject, math as score from row_scores
union all
select name, 'english' as subject, english as score from row_scores
union all
select name, 'bio' as subject, bio as score from row_scores;
```
---

# 分组内排序，并添加Row id

## 1. 准备

现在有一张消费金额表 userMoney ，内容如下：

month | name | money
---|---|---
01 | aaa | 1000
01 | bbb | 2000
01 | ccc | 3000
02 | aaa | 5000
02 | bbb | 2000
02 | ccc | 3000

## 2. 目标

我们想找出每个月里消费最多的两个人，以及它们的消费金额，在当月的排名。

预期结果如下：

month | name | money | rank
---|---|---|---
01 | ccc | 3000 | 1
01 | bbb | 2000 | 2
02 | aaa | 5000 | 1
02 | ccc | 3000 | 2

## 3. 语句

```sql
select * from (
select 
month, name, money, row_number() over (distribute by month sort by money desc) as rank
from userMoney
) as temp
where temp.rank < 3;

```
---

# 使用 Split 函数的单行变多行

## 1. 准备

有时候，我们会有类似下面的表：

id | values
---|---
1 | aaa,bbb,ccc

但是我们想得到如下的表：

id | v
---|---
1 | aaa
1 | bbb
1 | ccc

## 语句

```sql
select id, v from test lateral view explode(split(values,',')) adtable as v;  

```

# 使用 Split 函数的单行变多行

## 1. 准备

有时候，我们会有类似下面的表：

| id   | values      |
| ---- | ----------- |
| 1    | aaa,bbb,ccc |

但是我们想得到如下的表：

| id   | v    |
| ---- | ---- |
| 1    | aaa  |
| 1    | bbb  |
| 1    | ccc  |

## 2. 语句

```sql
select id, v from test lateral view explode(split(values,',')) adtable as v;  
```

# 组内排序后合并

## 1. 准备

输入表：

| id   | v    |
| ---- | ---- |
| 1    | cccc |
| 1    | aaaa |
| 1    | bbbb |

预期的输出表:

| id   | values         |
| ---- | -------------- |
| 1    | aaaa,bbbb,cccc |

这里想要它们先排序后再进行拼接操作。

## 2. 语句

```sql
SELECT id, CONCAT_WS(",", SORT_ARRAY(COLLECT_SET(v))) as values from t GROUP BY id
```

# 创建 Partition 或 bucket

## partition

```sql
create table state_part(District string,Enrolments string) PARTITIONED BY(state string);
```

## bucket

```sql
create table state_part(District string,Enrolments string) clustered by Enrolments into 4 buckets row format delimited fields terminated by ',';
```

详细参考[这里](https://www.guru99.com/hive-partitions-buckets-example.html)。