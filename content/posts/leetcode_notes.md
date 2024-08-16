+++
title = "刷题笔记"
author = ["colawithsauce"]
draft = true
+++

## 遍历类 {#遍历类}

遍历类的难点在哪里？我觉得我很多的时候搞不清下面几点：

1.  要用几个变量？
2.  变量的更新时机？有的时候，我们需要在下一轮循环中，更新上一轮循环所涉及概念的变量。
3.  开始与结束是不是要特殊处理？

其实遍历类的题目难点主要在于，要清楚地描述每个变量的实际意义，在思考过程中不要把他的意义给弄混了。就这么简单。而在整个遍历过程里，我们先思考遍历的中间是什么情况，再考虑开头与结尾的特殊处理。


## [2522. 将字符串分割成值不超过 K 的子字符串](https://leetcode.cn/problems/partition-string-into-substrings-with-values-at-most-k/) {#2522-dot-将字符串分割成值不超过-k-的子字符串}

首先，最容易想到的解法是暴力法。将所有的情况遍历，同时统计 segments 最小的值。用汉语描述如下：

1.  将字符串分成两部分，对第一部分的长度 i 进行遍历
    1.  如果前缀 s[0..i] 小于 k，则
        1.  返回 f(s[i..n], k) + 1
    2.  否则直接退出循环

使用贪心算法？即：在从左到右能够得到最大的值的时候就去得。

看了眼题解，确实是只要向右遍历就可以了。直接贪心可以秒。一次遍历，这个过程要想清楚。

1.  先得一位数，如果这个数小于，那就直接整个函数返回 -1
2.  再加一位数，若这个数大于等于 k，则累积最后的结果，并且从 1 开始。如果这个数小于 k，则继续得下一位数。


## [[257] 二叉树的所有路径](https://leetcode.cn/problems/binary-tree-paths/description/) {#257-二叉树的所有路径}

本题要给出从根节点到叶子节点的所有的路径（任意顺序），可以使用前序遍历法。

如何做呢？我们可以维护一个全局的path，表示当前所经过的路径，和结果向量vec；定义 visit 函数为更新path，且在当前节点是叶子的时候将 path 压入 vec

这道题卡了很久，因为我不知道 rust 的二叉树遍历怎么写。以为要很复杂的处理，其实也没有。

```rust
use std::cell::RefCell;
use std::rc::Rc;
use std::collections::VecDeque;

#[derive(Debug, PartialEq, Eq)]
pub struct TreeNode {
    pub val: i32,
    pub left: Option<Rc<RefCell<TreeNode>>>,
    pub right: Option<Rc<RefCell<TreeNode>>>,
}

impl TreeNode {
    #[inline]
    pub fn new(val: i32) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(TreeNode {
            val,
            left: None,
            right: None,
        }))
    }
}

fn my_preorder_rec(root: &Option<Rc<RefCell<TreeNode>>>) -> Vec<String> {
    let mut vec = vec![];
    let mut rst = vec![];
    my_preorder_rec_helper(root, &mut vec, &mut rst);
    rst
}

fn my_preorder_rec_helper(root: &Option<Rc<RefCell<TreeNode>>>, vec: &mut Vec<i32>, rst: &mut Vec<String>) {
    if let Some(x) = root {
        let node = x.borrow();
        vec.push(node.val);
        if node.left.is_none() && node.right.is_none() {
            let mut ss = vec![];
            vec.iter().for_each(|i| {
                ss.push(i.to_string());
            });

            rst.push(ss.join("->"));
        }
        my_preorder_rec_helper(&node.left, vec, rst);
        my_preorder_rec_helper(&node.right, vec, rst);
        vec.pop();
    }
}

fn main() {
    let root = Rc::new(RefCell::new(TreeNode::new(1)));
    root.borrow_mut().left = Some(Rc::new(RefCell::new(TreeNode::new(2))));
    root.borrow_mut().right = Some(Rc::new(RefCell::new(TreeNode::new(3))));
    root.borrow_mut().left.as_mut().unwrap().borrow_mut().left = Some(Rc::new(RefCell::new(TreeNode::new(4))));
    root.borrow_mut().left.as_mut().unwrap().borrow_mut().right = Some(Rc::new(RefCell::new(TreeNode::new(5))));

    let result = my_preorder_rec(&Some(root));
    println!("{:?}", result); // Output: [1, 2, 4, 5, 3]
}
```
