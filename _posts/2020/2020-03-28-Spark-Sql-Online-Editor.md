---
layout: post
title: Spark Sql 在线编辑器
date: 2020-03-28 13:39:04
author: admin
comments: true
categories: [Spark]
tags: [Big Data, Spark, Spark Sql, CodeMirror, Online Editor]
---

工作中 Spark Sql 占了不小的比重，为了提高开发效率，就想搞个在线编辑器，期待有语法高亮、语法检测、自动提示等功能。技术栈主要包含 CodeMirror 以及 Spark Catalyst。

<!-- more -->
---


* 目录
{:toc}
---

## 技术栈

### 1. JavaScript

没啥好说的，基础知识。入门可以到 [这里](https://www.w3schools.com/js/) 学习。

### 2. Vue

这个也是很火的一个前端框架。可以到官网进行了解。

### 3. CodeMirror
是一个非常流行的前端编辑器框架。自带了很多有用的功能，也支持自定义语言的拓展。

github上有个现有的项目可以和vue结合起来使用，叫做 [vue-codemirror](https://github.com/surmon-china/vue-codemirror)。

### 4. ANTLR4

是个非常有用的工具，可以使用语法文件（g4文件）定义语法，然后将语法文件生成对应的词法、语法解析器，以便其他语言调用。封装了复杂难懂的语法树生成过程，便于使用。详细信息到[官网](https://github.com/antlr/antlr4/blob/master/doc/getting-started.md#a-first-example)了解。



## 重点介绍

### 1. CodeMirror的使用

#### 1）注册新的语言类型

语言类型在 CodeMirror 称为 Mode。

CodeMirror 使用以下代码来定义一个语言类型：

```javascript
CodeMirror.defineMode(languageModeName, function(config, parserConfig) {.....})
```

具体例子可以参考 CodeMirror 的[source code](https://github.com/codemirror/CodeMirror/blob/master/mode/sql/sql.js#L14).

定义了新的mode后，就可以定义语言的子模块（MIME）：

```javascript
CodeMirror.defineMIME(specificLanguageModeName, config)
```

源码参考[这里](https://github.com/codemirror/CodeMirror/blob/master/mode/sql/sql.js#L452)。

#### 2）定义语法高亮

语法高亮（Syntax Highlighting）是在定义语言的时候实现的。可以设置 “keywords” 和 “builtins”的值， CodeMirror 会把它们自动注色。

#### 3）自定补全
自定补全（auto-complete）在 CodeMirror 被叫做 "hint"。 

```javascript
CodeMirror.registerHelper("hint", languageModeName, function(editor, options) {....})
```

当编辑器具有每个更改事件时，将调用此提示函数。 并且此提示函数接受 “editor” 和 "options" 作为两个参数，并返回带有选择列表和提示位置的对象（默认为光标位置）。

已经有多种语言的实现，我们可以阅读[它们](https://github.com/codemirror/CodeMirror/blob/master/addon/hint/sql-hint.js)以了解如何实现此功能。

#### 4）语法检测和错误显示

语法检查和显示错误功能在CodeMirror中称为“ lint”。

```javascript
CodeMirror.registerHelper("lint", languageModeName, function(text) {.....})
```

当编辑器发生更改事件时，将调用该lint函数。这个lint函数接受编辑器的文本作为一个参数，并返回一个错误列表，每个错误都有错误消息，错误消息的位置和严重性。

具体实现可以参考[这里](https://github.com/codemirror/CodeMirror/blob/master/addon/lint/)。


### 2. ANTLR4 的使用

#### 1）ANTLR4 IDEA plugin

首先需要添加ANTLR4 IDEA插件。 这个插件可以帮助我们更轻松地编写g4。这里直接沿用 Spark Sql 自带的语法文件：[Spark-Catalyst](https://github.com/apache/spark/tree/master/sql/catalyst)。

如果要实现自己的语法文件，可以参考[这里](https://github.com/antlr/antlr4/blob/master/doc/grammars.md )。

#### 2）从 g4 文件中生成 JS parser
因为我们想将ANTLR4与vue一起使用，所以我们需要将covent g4文件转换为JS文件。 我们可以使用npm软件包antlr4-tool代替将ANTLR4安装为软件。

#### 3）与 CodeMirror结合

这个不废话，直接上代码 [sparksql-lint.js](https://github.com/waltyou/spark-sql-online-editor/blob/master/src/sparkSql/sparksql-lint.js) 。



## 总结

项目地址：[spark-sql-online-editor](https://github.com/waltyou/spark-sql-online-editor) 。如有问题，欢迎issue。
