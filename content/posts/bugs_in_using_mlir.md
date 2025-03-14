+++
title = "记录一些写 MLIR 的时候踩的坑"
author = ["colawithsauce"]
date = 2025-03-14T17:45:00+08:00
draft = false
+++

## 在 Pattern 里面忘记写 `return success()` {#在-pattern-里面忘记写-return-success}

会导致报一个非常难以发现的栈错误，栈错误的位置在库函数调用完 matchAndRewrite 之后，也就是 xxxConversion.h 里面的 `return matchAndRewrite(...)` 后面报错。

这个问题很隐蔽，调了我半个多小时。
