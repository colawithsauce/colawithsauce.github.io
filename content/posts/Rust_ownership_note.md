+++
title = "Rust 所有权"
author = ["colawithsauce"]
tags = ["OwnerShip"]
categories = ["Rust"]
draft = true
+++

abstract: Rust 的所有权管理经常让我头痛。我这里写一些我关于这个主题的抽象层面的感悟，当然，也要补充案例分析。务必详细地分析各种所有权问题。


## 下面是我遇到的情况 {#下面是我遇到的情况}

1.  vec.push(x) 的时候会将 x 的所有权转移到 vec 里面去。其实任何的函数传参，只要不是引用类型传，那就会出现 move
2.  vec.pop() 的时候，会将所 pop 的元素转移出来。实际上，任何的函数返回，都会出现 move
3.  `if let Some(x) = abc` 会把 abc 的所有权转移
4.  `for i in vec` 会使用 move 语义，将 vec 消费掉（如果不想这样的话，显式使用 iter()，这是借用语义）
5.  有些函数会在 Document 里面讲，他 consume 了这个东西的所有权。这种时候所有权会发生转移。
6.  像链表的这种情况：
    ```rust
           let mut pioneer = head.unwrap().next;
           let temp = pioneer.unwrap().next;
    ```
    他会将 pioneer 和 head 的所有权给转移


## 什么时候可能会转移所有权？ {#什么时候可能会转移所有权}

所有权转移发生在任何：

1.  没有实现 Copy Trait 的类型
2.  没有显式指定 &amp;var 使用引用类型

的时候。也就是说，如果不指定，则必然会发生所有权转移


## TIPS {#tips}


### TIPS 默认无法转移 field 的所有权，只转移变量的所有权 {#tips-默认无法转移-field-的所有权-只转移变量的所有权}

比如上面提到的

```rust
let mut pioneer = head.unwrap().next;
let temp = pioneer.unwrap().next;
```

这种情况，会把 pioneer 和 head 的所有权给转移掉。


### TIPS 若需要转移 filed 所有权，使用 `take()` {#tips-若需要转移-filed-所有权-使用-take}

```rust
remain = n1.next.take();  // take()将n1打断，这样n1只有一个值，返回值是除n1节点外的剩余节点
                          // node.next是Option<T>
                          // take()是用默认值替换原有的值，所以n1.next就变为None
```


## 如何 think in rust? {#如何-think-in-rust}

不应该用“赋值”等原来的旧有的语言，而应该使用『所有权转移』，『借出』，『借用』的新语言来思考问题。
