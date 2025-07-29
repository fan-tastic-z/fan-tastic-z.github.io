---
title: 20250729-daily-rust
date: 2025-07-29 18:09:59
tags:
  - daily
  - rust
categories:
  - daily
published: true
---
## Rust Tips

Rust 中 iterators 非常强大，两个使用的函数：

- peekable: 允许查看下一个元素但不使用它
- partition(cond_fn): 根据条件将迭代器划分为单独的集合

```Rust
fn main() {
    let mut iter = vec![1, 2, 3].into_iter().peekable();
    let a = iter.peek();
    println!("{:?}", a);
    let b = iter.next();
    println!("{:?}", b);

    let (events, odds): (Vec<_>, Vec<_>) = (1..6).partition(|&x| x % 2 == 0);
    println!("{:?}", events); // [2, 4]
    println!("{:?}", odds); // [1, 3, 5]
}
```

## Rust 学习资源

<https://rust-exercises.com/> 非常不错的rust 新手学习资源，同时这个课程可以在 JetBrains 的 Rust IDE RustRover 中使用。<https://blog.jetbrains.com/education/2025/07/28/rust-exercises-rustrover/>
