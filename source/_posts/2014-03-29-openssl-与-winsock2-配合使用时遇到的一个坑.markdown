---
layout: post
title: "OpenSSL 与 WinSock2 配合使用时遇到的一个坑"
date: 2014-03-29 23:06:38 +0800
comments: true
tags:
  - openssl
  - windows
  - network
---

今天才发现，这里已经两年多没有被照看过了，估计连树都要长出来了吧。

在用 Windows 的 WSAEventSelect 模式进行网络编程时，比较固定的一个模式是这样的：

``` c
...
WSAEventSelect(sock, events[0], FD_READ);
...
while (1)
{
    ret = WSAWaitForMultipleEvents(1, events, FALSE, timeout, FALSE);
    if (ret >= WSA_WAIT_EVENT_0 && ret < WSA_WAIT_EVENT_0 + 1)
    {
        eventIndex = ret - WSA_WAIT_EVENT_0;
        if (eventIndex == 0)
        {
            WSAEnumNetworkEvents(sock, events[0], &networkEvents);
            if (networkEvents.lNetworkEvents & FD_READ)
            {
                ...
                recv(sock, buf, sizeof(buf));
                ...
            }
            ...
        }
        ...
    }
    ...
}
...
```

当要处理的 sock 属于一个 OpenSSL 连接时，只要把当中的 recv 换成 SSL\_read 就行了，当然前面还得加上些 readWaitonWrite 之类的标志位检查啥的，这里就不详细列举了，请参考《Network Security with OpenSSL》一书中的 5.2.2.3 小节。

但我在实际使用中（事实上是在压力测试中）发现，当程序运行一段时间之后，WSAWaitForMultipleEvents 就会忽然不再返回可读信号了，从而导致对该 socket 的接收操作完全停止。

经过各种调试手段，甚至是在 OpenSSL 中加入调试输出之后，终于发现问题出在 SSL\_read 的实现机制上，貌似 OpenSSL 实现了某种程度的读写缓冲（具体没细看），使得对 SSL\_read 的一次调用，并不一定会触发其对底层 socket 的读取操作。而如果没有对底层 socket 的读取操作，那么 windows 的对应 event 对象就不会被 reenable （参考 MSDN 中对 WSAEventSelect 接口的说明文档），从而导致 WSAWaitForMultipleEvents 不再对该事件作检查。

原因找到了，那么相应的解决方法也就不难发现了，对于我的使用场景来说（不完全是像上面的示例片断那么清晰），最简单的作法就是在 SSL\_read 之前加一个空的 recv 调用，其传给 recv 的第二、三个参数的值分别是 NULL 和 0 ，这样就能强制触发 windows event 的 reenable ，同时又不会影响到 SSL 对象内部的读取状态了。

PS. 遇到这个问题时，最重要的一步其实是如何能够稳定而快速的复现问题。一开始做压力测试时，只有在连续跑个一天以上时才会不定时的出现，导致效率很低；后来偶然发现，通过虚拟机搭建的受限环境，反而能很快复现问题。想来这也是另一种形式的压力测试吧，正常的压力测试是保持环境不变，加大请求压力；这里则变成了保持请求不变，同时压缩可用环境。
