+++
title = "MLIR 如何给 Op 增加 EnumAttr ？"
author = ["colawithsauce"]
date = 2024-10-31T02:49:00+08:00
tags = ["ATTACH"]
draft = false
+++

Debug 了很久，终于找到方法了。本来想着找到方法之后写篇文章细细道来，但是最后也只有三言两语好讲。

首先，查阅官方的资料发现，我们只需要在 td 文件里面这样写，就能给我们的 Op 增加一个 Attribute 参数了。
![](/ox-hugo/_20241031_023540screenshot.png)

但是试验之后发现并不可以，会提示 'MyEnumAttr' 不是 `mlir::my_dialect::` 命名域里面的成员。于是经过一段时间的与 CMake 和 C++ 编译报错信息的搏斗，终于让我找到问题所在了。

让我们开门见山吧：

首先，在 `CMakeLists.txt` 文件里，需要加上下面的➀、➁、➂，缺一不可：

{{< figure src="/ox-hugo/_20241031_024027screenshot.png" >}}

1.  LLVM_TARGET_DEFINITIONS 的作用：指定由什么文件来生成对应的声明与定义
2.  `-gen-enum-{decls,defs}` 的作用：生成这个 enum 的声明与定义
3.  `-gen-attrdef-{decls,defs}` 的作用：生成这个 enumAttr 的声明与定义

没错！enumAttr 他又是 enum 又是 attr。面对这种情况，MLIR 官方文档里面的[Attributes 教程](https://mlir.llvm.org/docs/DefiningDialects/AttributesAndTypes/#attributes)只能说是部分正确，因为他只告诉了我们 Attr 要怎么加。但是却没有告诉我们 EnumAttr 怎么加。

但是仅仅在告诉我们如何加 Attr 这方面，官方文档也并不全面。因为他没有告诉我们 `.h` 文件要怎么写。

{{< figure src="/ox-hugo/_20241031_024659screenshot.png" >}}

如上图所示，➃、➄两步想必大家已经很熟悉了。我们要注意两点：

1.  ➀ 要在 ➂ 和 ➄ 前，因为他们之间有一个全序的依赖关系。我后面困惑了几分钟的问题就是因为我把 ➂ 给放到 ➄ 下面了。
2.  不要忘记添加 ➁ 这一句。如果不添加这句，则不会 include 任何东西。我也在这里犯了错

EDIT1:
发现链接 xxx-opt 时提示 undefined reference，于是我发现需要在 Arith.cpp 里面增加：

{{< figure src="/ox-hugo/_20241031_040745screenshot.png" >}}

然后发现另一个坑，在编译的时候会报错：

{{< figure src="/ox-hugo/_20241031_041132screenshot.png" >}}

然后发现是我在 Arith.cpp 里面少引入了一个头文件：

```cpp
#include "llvm/ADT/TypeSwitch.h"
```

是的，如果不直接引入这个头文件就会报错。
