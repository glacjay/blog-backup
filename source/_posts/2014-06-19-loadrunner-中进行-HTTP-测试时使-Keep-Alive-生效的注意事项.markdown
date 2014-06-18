---
layout: post
title: "LoadRunner 中进行 HTTP 测试时使 Keep-Alive 生效的注意事项"
date: 2014-06-19 00:23:16 +0800
comments: true
tags:
  - web
---

工作需要，在使用 LoadRunner 进行 HTTP 测试时，为了使每个虚拟用户在不同的循环周期中都能保持长连接，则除了要打开 Keep-Alive 运行时配置（默认打开）之外，还有两个选项需要修改，这两个选项都在运行时配置的「Browser - Browser Simulation」中：

- Simulate browser cache ：取消勾选。禁用对浏览器缓存机制的模拟，令虚拟用户确实的每次都真正发起与 Web 服务器的对话，而不是只读取一下缓存。

- Simulate a new user on each iteration ：取消勾选。不要在每次执行 Action 之前重置虚拟用户的状态，不然会把原来的 TCP 长连接也重置掉，Keep-Alive 就没用了（更准确地说，是在不同循环周期之间就没用了）。
