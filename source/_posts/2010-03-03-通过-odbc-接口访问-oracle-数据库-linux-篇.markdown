---
categories:
  - Database
comments: true
layout: post
published: true
status: publish
tags:
  - oracle
title: "通过 ODBC 接口访问 Oracle 数据库 -- Linux 篇"
type: post
---

首先，你要安装好 UnixODBC 软件包，这个就不多说了。

然后，安装 Oracle 官方客户端，因为我的使用环境为 Fedora 12 ，所以我下载安装的是 `oracle-xe-client-10.2.0.1-1.0.i386.rpm` 。

装好之后，要设置一些环境变量，我是用的一个 Shell 脚本来完成这项工作的，你可以把它放在 `/etc/profile.d/` 目录下并加上可执行权限来让其在系统启动时自动执行，也可以直接运行这个脚本来使其立即生效。脚本如下：

``` bash
# File: oracle.sh
export ORACLE_HOME=/usr/lib/oracle/xe/app/oracle/product/10.2.0/client
export PATH=$PATH:$ORACLE_HOME/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME/lib
export TNS_ADMIN=$ORACLE_HOME/network/admin
export TWO_TASK=test
export NLS_LANG=”Simplified Chinese_china.UTF8”
export NLS_DATE_FORMAT="YYYY-MM-DD HH24:MI:SS"
```

其中 `TWO_TASK` 变量的值应设为 `tnsnames.ora` 文件中的那个键名（下面再讲）， `NLS_LANG` 是为了 UTF-8 编码，而 `NLS_DATE_FORMAT` 设的是 `DATE` 类型的数据在插入和显示时所用的格式，这里设的是我比较习惯的一种。

然后在 `$ORACLE_HOME` 目录中建一个子目录 `network/admin` ，并在其中新建两个文本文件如下：

``` plain
# File: sqlnet.ora
SQLNET.AUTHENTICATION_SERVICES = (NTS)
NAMES.DIRECTORY_PATH = (TNSNAMES, EZCONNECT)
```

这个文件中的都是一些固定的配置，其中 `TNSNAMES` 和 `EZCONNECT` 分别为我所用到的两种指定连接目标的方式。另一个文件：

``` plain
# File: tnsnames.ora
test =
  (DESCRIPTION =
    (ADDRESS =
      (PROTOCOL = TCP)
      (HOST = 192.168.0.2)
      (PORT = 1521)
    )
    (CONNECT_DATA =
      (SID = test)
    )
  )
```

第二行中的 `test` 就是 `TWO_TASK` 环境变量的值， `SID` 就是数据库名，其它的就不需要说明了。当然，在这个文件中可以配置多个数据库连接，用不同的名称来标识（ `TWO_TASK` 就是用来选择的），这个例子中只有一个配置。

以上的步骤主要是 Oracle 相关，都完成了之后，可以先测一下 Oracle 客户端是不是能正常使用了：

``` plain
$ sqlplus username/password@test
```

这个命令可以连接数据库。这种参数格式就是上面提到的 `TNSNAMES` 方式，而 `EZCONNECT` 方式则是下面的样子（这种方式就不需要在 `tnsnames.ora` 文件中进行配置了）：

```
$ sqlplus username/password@192.168.0.2:1521/test
```

成功之后可以试试下面的 SQL 语句：

```
> select * from some_table;
```

接下来就是配置 UnixODBC 了。首先是 `odbcinst.ini` 文件：

``` ini
[oracle]
Driver = /usr/lib/oracle/xe/app/oracle/product/10.2.0/client/lib/libsqora.so.10.1
```

然后是 `odbc.ini` 文件：

``` ini
[oracle]
Driver = oracle
```

都没什么可配的。

最后就是程序了，跟其它的 ODBC 程序没什么不同，除了连接字符串，需要写成这个样子：

``` plain
"DSN=oracle;UID=username;PWD=password"
```

`DSN` 就是 ODBC Data Source 名，也就是上面 `odbc.ini` 文件中方括号内的名字； `UID` 就是用户名， `PWD` 就是密码。

好了，就这么多。

PS. 我和另一位同事分别碰到过一次一个特别令人头痛的问题，就是在连接时， UnixODBC 说 `libsqora.so.10.1` （就是在 `odbcinst.ini` 文件中指定的那个）找不到，虽然这个文件明明就在那边；用 `ldd` 命令查看该文件的结果是说“这不是一个动态可执行文件”。更不幸的是，我不知怎么在我的机器上把这个问题给莫名其妙地解决了，却想破头都不知道是怎么解决的。太失败了。

-----

**Update:** 问题解决了，出问题的都是新装的 Fedora 12 机器，后来做了次全系统的更新，再建个链接，就好了。只是不知道倒底是哪个软件包的版本太老，试了 `glibc` 、 `libtool` 、 `kernel` 都不是。

**Update 2012-12-31:** 找到问题的原因了，都是 SELinux 惹的祸，关之即可，或者配下相关的策略啥的。
