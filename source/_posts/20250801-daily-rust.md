---
title: 20250801-daily-rust
date: 2025-08-01 17:10:15
tags:
  - daily
  - rust
categories:
  - daily
---
## Rust Tips

Rust 2024: let chains in if and while

```Rust
fn main() {
    let numbers = [10, 20, 30, 40];
    match sum_first_two(&numbers) {
        Some(sum) => println!("The sum of the first two numbers is: {}", sum),
        None => println!("Not enough numbers to sum."),
    }
}

fn sum_first_two(nums: &[u8]) -> Option<u8> {
    let mut iter = nums.iter();
    if let Some(first) = iter.next()
        && let Some(second) = iter.next()
    {
        // Use checked_add to avoid overflow
        return first.checked_add(*second);
    }
    None
}
```

## Rust 学习资源

- <https://www.howtocodeit.com/articles/master-hexagonal-architecture-rust>: Hexagonal Architecture in Rust
