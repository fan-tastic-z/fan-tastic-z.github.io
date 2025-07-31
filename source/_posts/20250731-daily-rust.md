---
title: 20250731-daily-rust
date: 2025-07-31 14:48:58
tags:
  - daily
  - rust
categories:
  - daily
published: true
---
## Rust Tips

`drain` 从 vector 批量删除指定范围，并以迭代器的形式返回所有移除的元素。

```Rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5];
    let evens: Vec<_> = numbers.drain(..).filter(|&x| x % 2 == 0).collect();
    println!("Even numbers: {:?}", evens); // Even numbers: [2, 4]
    println!("Remaining numbers: {:?}", numbers); // Remaining numbers: []
}
```

## AI 趣事

<https://x.com/dotey/status/1950733173714756042>

![Vibe Coding](/images/Vibe-Coding.jpg)
