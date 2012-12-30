---
categories:
  - Compiler
comments: true
layout: post
published: true
status: publish
tags:
  - bison
  - compiler
title: "Bison 中的 Prologue 的格式变迁"
type: post
---

今天在用 Bison 手册中的 C++ 例子作为一个编译器前端实现的起点时发现，这个官方示例居然编译通不过，具体错误为在 Bison 生成的头文件中，没有 Driver 类的声明。按照 POSIX Yacc 标准，位于 %union 块之前的 Prologue 区中的代码，应该会被拷到头文件中的啊，于是 google 半天，在某个地方的 Bison ChangeLog 中找到了线索。

原来，POSIX Yacc 确实应该符合上述行为，可是 Bison 这家伙为了统一性，从 2.3a 版开始，就把所有 Prologue 中的代码，不管是 %union 前的还是之后的，统统只拷到实现文件中而不管头文件了。而为了对应不同的需求，又新增了一套 Prologue 语法，就是 %before-header 等四个新的 directive。好吧，这样也不错，以显式的统一格式的声明代替了可能会让人抓狂的隐规则。遂试之，结果告诉我说语法错误。接着往上看 ChangeLog 才发现，到了 2.3b 就又改了，把 %before-header 改成了 %code 之类的。这回终于没问题了。

我说，这也太不厚道了吧，这种兼容性改动，我在它的文档里面扒了半天都没看到半个字，而且还是出现在流行度这么高的软件中。于是深刻体会到“错文档不如无文档”的道理啊。由此看来，要做好软件还是需要有相当的责任感的啊。
