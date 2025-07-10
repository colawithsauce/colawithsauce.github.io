+++
title = "给 reMarkable 添加中文字体"
author = ["colawithsauce"]
date = 2025-07-10T16:10:00+08:00
draft = false
+++

## 使用 USB 线连接并上传中文字体 {#使用-usb-线连接并上传中文字体}

```shell
scp xxx.ttf 10.11.99.1:~/
```

密码在 reMarkable 的 Settings &gt; Help &gt; Copyrights and licences 里面，倒数第二段。用加黑字体标注的那个。这里会用到，下面 ssh 的时候也会用到。


## 把字体链接到系统的字体目录 {#把字体链接到系统的字体目录}

```shell
ssh 10.11.99.1
```

登陆后，

```shell
ln -sf ~/xxx.ttf /usr/share/fonts/ttf/xxx.ttf
```


## 常见问题 {#常见问题}


### 为啥我一更新字体就消失了？ {#为啥我一更新字体就消失了}

当然，一更新 / 目录就会被重新安装，所有的修改都会被消除。目前的办法就是每次更新完就用上面这条命令更新一下。


### 不能够放到 `/home/root/` 里面的 font 目录吗？ {#不能够放到-home-root-里面的-font-目录吗}

不行，亲测不行。可能是 `xochitl.service` 不会去读我们的 `/home/root/` 目录。而且这里折腾他去读我们的目录也是没意义的，因为即使读，系统更新也会消除此修改。
