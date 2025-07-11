+++
title = "记一次 distcc 运维"
author = ["colawithsauce"]
date = 2025-07-05T21:03:00+08:00
draft = false
+++

## 安装阶段 {#安装阶段}

```shell
sudo apt update
sudo apt install distcc
```


## 配置基础的内容 {#配置基础的内容}

按照所需要的配置了一番

```shell
# /etc/default/distcc

# Defaults for distcc initscript
# sourced by /etc/init.d/distcc

#
# should distcc be started on boot?
#
# STARTDISTCC="true"

STARTDISTCC="true"

#
# Which networks/hosts should be allowed to connect to the daemon?
# You can list multiple hosts/networks separated by spaces.
# Networks have to be in CIDR notation, e.g. 192.168.1.0/24
# Hosts are represented by a single IP address
#
# ALLOWEDNETS="127.0.0.1"

ALLOWEDNETS="192.168.1.0/24 0.0.0.0 127.0.0.1"

#
# Which interface should distccd listen on?
# You can specify a single interface, identified by it's IP address, here.
#
# LISTENER="127.0.0.1"

LISTENER="0.0.0.0"

#
# You can specify a (positive) nice level for the distcc process here
#
# NICE="10"

NICE="10"

#
# You can specify a maximum number of jobs, the server will accept concurrently
#
# JOBS=""

JOBS="10"

#
# Enable Zeroconf support?
# If enabled, distccd will register via mDNS/DNS-SD.
# It can then automatically be found by zeroconf enabled distcc clients
# without the need of a manually configured host list.
#
# ZEROCONF="true"

ZEROCONF="true"
```


## 问题出现 {#问题出现}


### 本机始终无法编译成功 {#本机始终无法编译成功}

我在本机使用下面的命令尝试 distcc，发现始终无法编译成功。

```shell
DISTCC_HOSTS="127.0.0.1:3632" DISTCC_VERBOSE=1 distcc gcc -c /tmp/test.cpp -o /dev/null
```

查看 `tail -f /var/log/distccd.log` ，发现异常字段：

```text
distccd[490557] (dcc_check_compiler_whitelist) CRITICAL! x86_64-linux-gnu-gcc not in /usr/lib/distcc or /usr/lib/distcc whitelist.
```

怀疑是我的白名单写得有问题，原本的白名单 `/etc/distcc/commands.allow.sh` 里面是：

```shell
#!/bin/sh
# --- /etc/site/current/distcc/commands.allow.sh ----------------------
#
# This file is a shell script that gets sourced by /etc/init.d/distcc.
# It's purpose is to optionally set the following environment
# variables, which affect the behaviour of distccd:
#
#     DISTCC_CMDLIST
#         If the environment variable DISTCC_CMDLIST is set, distccd will
#         load a list of supported commands from the file named by
#         DISTCC_CMDLIST, and will refuse to serve any command whose last
#         DISTCC_CMDLIST_MATCHWORDS last words do not match those of a
#         command in that list.  See the comments in src/serve.c.
#
#     DISTCC_CMDLIST_NUMWORDS
#         The number of words, from the end of the command, to match.  The
#         default is 1.
#
# The interface to this script is as follows.
# Input variables:
#    CMDLIST: this variable will hold the full path of the commands.allow file.
# Side effects:
#    This script should write into the commands.allow file specified by
#    $CMDLIST.  It should write the list of allowable commands, one per line.
# Output variables:
#    DISTCC_CMDLIST and DISTCC_CMDLIST_NUMWORDS. See above.
#-----------------------------------------------------------------------------#

# Here are the parts that you may want to modify.

numwords=1
allowed_compilers="
  /usr/bin/cc
  /usr/bin/c++
  /usr/bin/c89
  /usr/bin/c99
  /usr/bin/gcc
  /usr/bin/g++
  /usr/bin/*gcc-*
  /usr/bin/*g++-*
"

# You shouldn't need to alter anything below here.

[ "$CMDLIST" ] || {
   echo "$0: don't run this script directly!" >&2
   echo "Run /etc/init.d/distcc (or equivalent) instead." >&2
   exit 1
}

echo $allowed_compilers | tr ' ' '\n' > $CMDLIST
DISTCC_CMDLIST=$CMDLIST
DISTCC_CMDLIST_NUMWORDS=$numwords
```

关键在 `allowed_compilers` 这里，发现 `x86_64-linux-gnu-gcc` 是不会被这些项目匹配到的。因此需要再加两行：

```shell
allowed_compilers="
  /usr/bin/cc
  /usr/bin/c++
  /usr/bin/c89
  /usr/bin/c99
  /usr/bin/gcc
  /usr/bin/g++
  *gnu-gcc
  *gnu-g++
"
```

这样就能够编译成功了。


### gentoo 上面无法编译成功 {#gentoo-上面无法编译成功}

在我的 gentoo 机器上面试图使用这个 host，发现还是不行。查看 log 发现还是有这样的行：

```text
distccd[490557] (dcc_check_compiler_whitelist) CRITICAL! x86_64-pc-linux-gnu-gcc not in /usr/lib/distcc or /usr/lib/distcc whitelist.
```

仔细比对可以发现，其实是因为 gentoo 发送给 distccd 的编译器名字里面多了一个 `-pc` 而我的 ubuntu 服务器上面是没有这个 `pc` 的。解决方法也很简单，只要让 ubuntu 能找到这个名字就好了。分两步走：

加白名单

```shell
sudo ln -sf ../../bin/distcc /usr/lib/distcc/x86_64-pc-linux-gnu-gcc
sudo ln -sf ../../bin/distcc /usr/lib/distcc/x86_64-pc-linux-gnu-g++
```

加 bin 目录

```shell
sudo ln -sf /usr/bin/x86_64-linux-gnu-gcc /usr/local/bin/x86_64-pc-linux-gnu-gcc
sudo ln -sf /usr/bin/x86_64-linux-gnu-g++ /usr/local/bin/x86_64-pc-linux-gnu-g++
```


### 远程版本与本地版本不一致 {#远程版本与本地版本不一致}

在我的 host 侧，默认的 gcc 版本是 gcc-13，但是 guest 侧是 gcc-14。按照 gentoo 官网上面的说法[^fn:1]，不匹配会导致问题，因此我配置了一下 host 侧的编译套件设置：

```shell
sudo apt install gcc-14 g++-14
```

并且让 gcc 链接到更高的版本

```shell
sudo ln -sf /usr/bin/x86_64-linux-gnu-gcc-14 /usr/local/bin/x86_64-pc-linux-gnu-gcc
sudo ln -sf /usr/bin/x86_64-linux-gnu-g++-14 /usr/local/bin/x86_64-pc-linux-gnu-g++
```

这样，编译就没有潜在的问题了。


## UPDATE：同时支持 ipv4 与 ipv6 {#update-同时支持-ipv4-与-ipv6}

把 `/etc/default/distcc` 更新一下就好了：

```shell
# xxx 改成自己的 ipv6 地址前缀
ALLOWEDNETS="::ffff:192.168.1.0/24 ::ffff:100.122.154.1 127.0.0.1 xxxx:xxxx:xxxx:xxxx::/64"

LISTENER="::"
```

效果如下：

```text
distccd[785370] (dcc_job_summary) client: ::ffff:192.168.1.3:57832 COMPILE_OK exit:0 sig:0 core:0 ret:0 time:36ms x86_64-pc-linux-gnu-gcc main.c
```

原来 ipv6 是可以兼容 ipv4 的……

[^fn:1]: <https://wiki.gentoo.org/wiki/Distcc> 见 Mixed GCC Verions 一节
