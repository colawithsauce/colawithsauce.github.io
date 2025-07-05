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
