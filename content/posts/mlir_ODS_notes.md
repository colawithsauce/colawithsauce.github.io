+++
title = "MLIR ODS 笔记"
author = ["colawithsauce"]
tags = ["ODS"]
categories = ["technote"]
draft = true
+++

Abstract: 这篇笔记的愿景是：首先，我将 MLIR ODS 的知识粗略地介绍一下。然后再给出我使用 MLIR ODS 定义一个 Op 的例子。我想，这就应该能给读者一些启发了。

MLIR 里面的 Constraints 分成三类：One Element Constraint, Multi Element Constraint, Trait

其中第一类可以理解成类型（Type），他分成元素的类型与Attributes的类型。第二类是多个元素的 Constraint。比如说需要表示多个参数都是同样的类型；第三个是表示这个Op本身的性质，比如说 Pure。这样这些Op就能够被别的程序利用。


## Predication {#predication}

一个 Predication 只能是下面两种之一：

1.  `CPred<"...">` 表达式，用来表达一个最小的 Predication
2.  把多数 `CPred<"...">` 用逻辑串并联起来的组合 Predication

他使用 `$_self` 占位符来表示最后断言作用的对象，用其它的或内置或自定义的 C++ 函数来限定之。


## Type&lt;CPred&gt; {#type-cpred}

ODS 里面的类继承关系是：Type -&gt; TypeConstraint -&gt; Constraint，我们使用 Type 的时候，使用的是 Type&lt;Pred&gt;，其中 Pred 是一个断言，而整个 `Type<Pred>` 则是一个类型。我们在编程的时候，要特别注意这里的类型与断言不能够搞混了。
