+++
title = "wine-wechat 中文乱码问题"
author = ["colawithsauce"]
date = 2024-03-09T01:06:00+08:00
categories = ["article"]
draft = false
+++

最近几天使用 wine-wechat 的时候发现我在输入框里面输入文字的时候，中文是豆腐块。但是发出之后以及其它的地方，中文又能够正常显示。省流：原来是 LC_ALL 惹的祸：

```sh
env LC_ALL="zh_CN.utf8" LANG="zh_CN.utf8" wechat
```

这样就能够正常显示中文了。我们想要将这个效果持久化，就将其写入 `wine-wechat.desktop` 中就好了：

```text
[Desktop Entry]
Exec=env WINEDEBUG=-all LC_ALL="zh_CN.utf8" LANG="zh_CN.utf8" wechat
# ...
```

这个问题我排查了很久，首先我想到可能是因为富文本编辑器依赖没有安装完全，所以我用 winetricks 安装了所有的富文本编辑器依赖项。后面又觉得可能是我刚好没有安装显示输入框里面中文所需要的字体，但是我明明在前几天还能够正常地使用的。于是我觉得可能是我乱添加依赖项，改动了注册表，导致显示的字体从原本有的字体被替换成了不存在的字体导致中文字体显示失败。于是我改了注册表，但是仍然有问题。最后我看论坛里面有人说首先要排查 `locale -a` 中是否有 `zh_CN.utf8` ，我灵光一闪才想到这其中关节。
