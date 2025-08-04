---
title: 20250804-daily-rust
date: 2025-08-04 17:17:39
tags:
  - daily
  - rust
categories:
  - daily
---
## Rust Tips

一种非常方便的方法来过滤 `HashMap` 数据

```Rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert("Alice", 80);
    scores.insert("Bob", 90);
    scores.insert("Charlie", 60);

    scores.retain(|_, &mut score| score > 60);
    println!("Scores after retain: {:?}", scores); // {"Bob": 90, "Alice": 80}
}
```
