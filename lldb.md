---
title: lldb 技巧
date: 2021-03-08
tags:
  - lldb
  -  技巧
---

[toc]

## 入门类知识

### lldb 入门文档

[与调试器共舞 - LLDB 的华尔兹](https://objccn.io/issue-19-2/)

### lldb 工具库推荐列表

* [DerekSelander](https://github.com/DerekSelander/LLDB)
* [chisel](https://github.com/facebook/chisel)

## lldb 中级文档

lldb 自定义命令开发相关的文章
[酷酷的哀殿 lldb 合集](https://ai-chan.top/tags/lldb)

### lldb 深入研究

底层机制相关知识或者文章

### lldbinit 文件

问题描述：部分用户可能遇到 `~/.lldbinit` 文件失效的情况。

问题分析：除了常见的 `~/.lldbinit` 文件外，本地可能存在其它**更高优先级**的配置文件导致 `~/.lldbinit` 文件失效

解决方案：参照以下顺序，将配置信息放到合适的文件

`lldb` 启动时，会依次尝试查找以下初始化文件

程序指定：

```sh
~/.lldbinit-lldb // 命令行
~/.lldbinit-Xcode  // Xcode
```

语言指定：

```sh
.lldbinit-<language>-repl // (比如， `.lldbinit-swift-repl`)
```

默认路径：

```sh
~/.lldbinit
```
