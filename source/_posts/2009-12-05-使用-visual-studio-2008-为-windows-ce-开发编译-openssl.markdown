---
categories:
  - Programming
comments: true
layout: post
published: true
status: publish
tags:
  - c++
  - windows
title: "使用 Visual Studio 2008 为 Windows CE 开发编译 OpenSSL"
type: post
---

我的编译环境是 Visual Studio Team System 2008 版本 9.0.21022.8 RTM ， Windows Mobile 5.0 SDK R2 （ VS2008 自带的版本）。当然， Perl 也是需要的，我装的是 ActivePerl 。我要编译的 OpenSSL 版本是 0.9.8e 。

## VS2008 的安装

那个 Web Developer Tools （好像叫这个）的安装会失败，又不能不装，根据网上的说明，要把它的目录单独从光盘上解压出来安装，且安装的时候要挂 Office 的安装光盘（我的 Office 版本是 2007 ）。这个装好了之后，再从光盘上安装 VS2008 就没有问题了。

## 配置编译环境

首先要写个定义环境变量的脚本，以把 CE 的路径给配置好。 VS2008 的那个命令行启动脚本只配置了 VC 的环境变量，没有配 CE 的环境变量，因为这个是要根据用户需要来做的。我写的脚本内容如下，你可能要根据自己的 VS2008 相关安装目录做调整。

``` bat
set PATH=D:\Program Files\Microsoft Visual Studio 9.0\VC\ce\bin\x86_arm;%PATH%
set INCLUDE=D:\Program Files\Microsoft Visual Studio 9.0\VC\ce\include;C:\Program Files\Windows Mobile 5.0 SDK R2\PocketPC\Include\Armv4i;D:\Program Files\Microsoft Visual Studio 9.0\VC\ce\atlmfc\include;%INCLUDE%
set LIB=C:\Program Files\Windows Mobile 5.0 SDK R2\PocketPC\Lib\ARMV4I;D:\Program Files\Microsoft Visual Studio 9.0\VC\ce\atlmfc\lib\armv4i;D:\Program Files\Microsoft Visual Studio 9.0\VC\ce\lib\armv4i;%LIB%

set OSVERSION=WCE501
set TARGETCPU=ARMV4I
set PLATFORM=VC-CE
set WCECOMPAT=D:\program\wcecompat
```

其中 `PATH` 、 `INCLUDE` 、 `LIB` 都是 CE 相关的环境变量，在编译 WCECompat 和 OpenSSL 的时候都是需要的，后面的是 WCECompat 需要的，其中有些是 OpenSSL 也需要的。

然后启动 Visual Studio 2008 命令提示（开始菜单），并运行上面的这个脚本，编译环境就算是准备好了，可以开始编译 WCECompat 了。

## 编译 WCECompat

要编译 CE 版本的 OpenSSL 的话，需要 WCECompat 这个库，这个库实现了许多在 Desktop 开发环境中有而在 CE 中没有的功能，而这些功能是编译与使用 OpenSSL 所必需的。

不过，官方版本的 WCECompat 现在已经用不起来了（虽然我没有进一步确定），我们要使用 OpenSSL 开发团队 fork 的分支，其下载地址在[http://github.com/mauricek/wcecompat](http://github.com/mauricek/wcecompat)，在这个页面上有打包下载的链接，我下载的版本是 cb796f5 （ git 是用 GUID 来表示版本的）。

下载下来之后，先解压，我的解压路径是 `D:\program\wcecompat` ，并进入该目录。这里注意，根据 WCECompat 的文档说明，不能把它解压到一个含空格的路径底下，不然会报文件找不到之类的错，这是因为 WCECompat 的 `makefile` 文件没有用引号括住文件名，而空格会把一个文件名分成两个。

然后要修改一个源码文件 `src/time.c` ，在其中找到函数 `_tzset` 的定义，这是一个空函数，把它整个注释掉（或删掉），不然在后面我们的程序要链接这个库的时候，会报这个函数重复定义的错。

运行命令

```
D:\program\wcecompat>perl config.pl
```

这样会生成需要的 `makefile` 文件，不过还不能直接用它来编译，要手动改一下，删掉或注释掉其中的 `src/winmain.cpp \` 这一行，不然后面链进我们的程序时会报找不到 WinMain 所需的 `main` 函数这个错（大概是这样说的）。

然后运行命令

```
D:\program\wcecompat>nmake
```

开始编译。完成之后，就会在 `lib` 目录中生成两个库文件 `wcecompat.lib` 和 `wcecompatex.lib` ，都是静态链接库。

## 编译 OpenSSL

还是同一个命令行窗口，进入 OpenSSL 的解压目录，我的是 `D:\program\openssl-0.9.8e` 。根据 `INSTALL.WCE` 文件中的说明，运行下列命令进行编译：

```
D:\program\openssl-0.9.8e>perl Configure VC-CE
...
D:\program\openssl-0.9.8e>ms\do_ms.bat
...
```

这一步完成之后，打开生成的 `ms\ce.mak` 文件，把第 `19` 行 `CFLAG` 的变量定义中的 `/WX` 选项给删掉，不然后面有编译警告会被当成错误，从而编译失败。

```
D:\program\openssl-0.9.8e>nmake -f ms\ce.mak
...
```

这一步的编译过程最终还是会失败退出，不过不要紧（也许吧），失败的是测试程序，这时看 `out32_ARMV4I` 目录中，已经有编译好的 `libeay32.lib` 和 `ssleay32.lib` 这两个文件了。因为前面是用 `ms\ce.mak` 文件而不是 `ms\cedll.mak` ，所以编译出来的这两个文件都是静态链接库。

## 测试 OpenSSL

最后来测试一下我们编译出来的 OpenSSL （以及 WCECompat ）。打开 VS2008 ，新建个智能设备的项目。然后打开项目属性对话框，先在“配置属性-> C/C++ ->常规”的“附加包含目录”中把 `D:\program\openssl-0.9.8e\inc32` 给加进来（测试的话加绝对路径就好了，实际开发的时候要把所有需要的文件拷到项目目录里面）（另，不要加 WCECompat 的 `include` 目录，不然会有 `abs` 函数不属于 `global namespace` 的编译错误），在“配置属性->链接器->常规”的“附加库目录”中把 `D:\program\openssl-0.9.8e\out32_ARMV4I` 和 `D:\program\wcecompat\lib` 给加进来，在“配置属性->链接器->输入”的“附加依赖项”中加入 `wcecompat.lib wcecompatex.lib libeay32.lib ssleay32.lib`。

打开个源文件（比如某个事件处理函数定义的地方），在最开始的地方加进去 OpenSSL 的头文件包含

``` c
#define NO_SYS_TYPES_H
#include <openssl/ssl.h>
```

其中第一行的宏定义是需要的。虽然 WCECompat 为我们提供了 `sys/types.h` 头文件，不过我们不能用（理由见上）。

然后在事件处理函数中加条语句

``` c
SSL_CTX *ctx = SSL_CTX_new(SSLv23_method());
```

编译下试试吧，看有没有问题。如果没有，那就恭喜了，我们暂时解决了将 OpenSSL 用于 CE 开发的问题，我还不能保证后面不会出其他的问题。

## 另

虽然我之前配置环境变量等步骤都是基于 PocketPC 来做的，不过最后编出来的库貌似也能用到 Smartphone 程序中，不知道会不会有什么问题。 Mobile 开发果然还是相当的麻烦啊。
