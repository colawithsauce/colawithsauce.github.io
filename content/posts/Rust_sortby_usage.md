+++
title = "Rust 的 sort_by 应该如何使用？"
author = ["colawithsauce"]
date = 2024-08-20T18:15:00+08:00
tags = ["SortBy"]
categories = ["Rust"]
draft = false
+++

Abstract: SortBy 取一个 `std::cmp::Ordering` 对象作为自己的参数，但是关于这一点，我有几个问题。

1.  `vec.sort_by(|a, b| b.ge(a))` 会是从小到大，还是从大到小？这个规律是什么？能否有办法快速地找到？这个很重要。
2.  `cmp::Ordering` 如何能够支持复合判别条件？比如我的 vec 是一个两元组，按其中的第一维顺序排序，而当第一维的两个元素偏序关系相同的时候，再把这两个元素按第二维逆序排序。这种需求如何在 rust 的 sort_by 函数里实现？

本文最初的设想就是回答上面这两个问题，并给出一些例子。

ANSWER: 这个问题其实并不复杂，我上网找 stack overflow 找到了[相关回答](https://stackoverflow.com/questions/67335967/how-to-combine-two-cmp-conditions-in-ordcmp)，它里面说到，我们可以使用 `.then()` 来实现

```rust
    fn cmp(&self, other: &Self) -> Ordering {
        self.id.cmp(&other.id)
            .then(self.age.cmp(&other.age))
    }
```

下面是官网关于 `then()` 和 `then_with()` 的介绍：<https://doc.rust-lang.org/std/cmp/enum.Ordering.html#method.then>

然后是这两个的例子

```rust
#[derive(Debug)]
struct Student {
    id : u32,
    name : String,
    age : u32,
}

let mut vec = vec![ Student {
    id: 1, name: String::from("Li Hua"), age: 18,
},
 Student {
    id: 3, name: String::from("Liu Huan"), age: 19,
},
 Student {
    id: 2, name: String::from("Liu HongTao"), age: 19,
}];

vec.sort_by(|a, b| a.age.cmp(&b.age).then(a.id.cmp(&b.id)));
println!("{:?}", vec);
```

注意比较函数会要求一个借用的参数，毕竟他不需要所有权转移才能工作。

另外，关于第一个问题，这个用法就是错的。不应使用 b.ge(a)，而应该使用 b.cmp(a)；若是这样，则从大到小排序。若是 a.cmp(b) 则从小到大排序。
