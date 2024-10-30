+++
title = "MLIR lit 测试报错 AttributeError: 'NoneType' object has no attribute 'use_lit_shell'"
author = ["colawithsauce"]
date = 2024-08-26T16:30:00+08:00
tags = ["Lit"]
categories = ["MLIR"]
draft = false
+++

最近整 MLIR 相关任务，FileCheck + Lit 是其 IR 转换的标准测试工具链。

但是我在图示目录树

```nil
project          # 的这个目录下
 |-lib
 |-include
 |+test
  |-lit.cfg.py
  |- ...
 |- ...
```

运行 `lit ./test/` 进行测试，报错 `AttributeError: 'NoneType' object has no attribute 'use_lit_shell'`

后来上网查，发现是因为我们运行测试的目录错了。我们不应该在 project/test 目录下面运行，这是源代码的目录。我们应该在目标目录下面运行。也就是说，如果我的编译目标目录是 `project/build` ，那我就应该 `lit project/build/test/`
