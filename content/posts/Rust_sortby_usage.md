+++
title = "Rust 的 sort_by 应该如何使用？"
author = ["colawithsauce"]
tags = ["SortBy"]
categories = ["Rust"]
draft = true
+++

Abstract: SortBy 取一个 `std::cmp::compare` 对象作为自己的参数，但是关于这一点，我有几个问题。

1.  `vec.sort_by(|a, b| b.ge(a))` 会是从小到大，还是从大到小？这个规律是什么？能否有办法快速地找到？这个很重要。
2.  `cmp::compare` 如何能够支持复合判别条件？比如我的 vec 是一个两元组，按其中的第一维顺序排序，而当第一维的两个元素偏序关系相同的时候，再把这两个元素按第二维逆序排序。这种需求如何在 rust 的 sort_by 函数里实现？

本文最初的设想就是回答上面这两个问题，并给出一些例子。
