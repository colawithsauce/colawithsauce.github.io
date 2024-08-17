+++
title = "Rust 笔记"
author = ["colawithsauce"]
draft = true
+++

## 所有权 {#所有权}

什么时候可能会转移所有权？

1.  vec.push(x) 的时候会将 x 的所有权转移到 vec 里面去
2.  `if let Some(x) = abc` 会把 abc 的所有权转移
3.  `for i in vec` 会使用 move 语义，将 vec 消费掉（如果不想这样的话，显式使用 iter()，这是借用语义）
