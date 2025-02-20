+++
title = "Linux 的一些小技巧"
author = ["colawithsauce"]
date = 2025-02-20T11:09:00+08:00
draft = false
+++

## Docker {#docker}

可以把 docker 的使用分成两种

1.  当作脚本使用
2.  当作终端使用

在做开发时的大部分情况下，使用 docker 是第二种使用方式。这个时候注意一件事情，使用 `docker exec -it container_name /bin/bash` 来 attatch 到 docker 而不是 `docker attatch`

因为 docker 启动的方式一般是 `docker run xxx -it /bin/bash` ，其会启动一个命令 `/bin/bash` ，如下图所示：

```nil
- /bin/bash *
```

若 attatch，那就会 attatch 到这个根命令。

1.  若退出，那就会导致 docker 的根命令退出，从而 docker 容器退出。
2.  若在 host 里面 attatch 多次（通常是出于想在这个 docker 容器里面跑多个耗时操作），第二次的 attatch 会挤掉第一次的 attatch。

但如果使用 `docker exec -it container_name /bin/bash` 的方式，就相当于在根命令下面再新建一个新的子进程。也就是说下面这样：

```nil
- /bin/bash
   +-- /bin/bash *
```

这样做就能回避上面提到的两个问题：

1.  若退出，也只是退出子进程，不会使得根进程退出。
2.  若多次 `exec` ，也只会是新建多个子进程，进程之间不会相互打扰，如下示意图：
    ```nil
    ​   - /bin/bash
          +-- /bin/bash
          +-- /bin/bash *
    ```

唯一的问题是，不能够再 `attatch` ，否则 `attatch` 后退出会导致根进程退出，从而整个 docker 容器被摧毁，如下图：

```nil
- /bin/bash * <- (exit)
   +-- /bin/bash
   +-- /bin/bash

----->

- /bin/bash (destroyed)
   +-- /bin/bash (destroyed)
   +-- /bin/bash (destroyed)
```


## tmux {#tmux}

-   使用 `tmux new -t name` 新建一个 tmux 实例，名字为 `name` 。
-   在 tmux 终端中，按下 `ctrl-a` 再(在 500 毫秒内)按下 `d` ，离开 tmux 终端（终端里面的命令仍然会在后台跑）。
-   使用 `tmux a -t name` 重新连接名为 `name` 的终端。
