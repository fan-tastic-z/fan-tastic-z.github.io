---
title: 20250730-daily-rust
date: 2025-07-30 19:56:05
tags:
  - daily
  - rust
categories:
  - daily
published: true
---
## Rust Tips

```Rust
fn main() {
    my_assert(false);
}

#[track_caller]
fn my_assert(condition: bool) {
    if !condition {
        let caller = std::panic::Location::caller();
        panic!(
            "Assertion failed at {}:{}:{}",
            caller.file(),
            caller.line(),
            caller.column(),
        );
    }
}
```

Rust中的`track_caller`宏,可以帮我们精准定位代码调用点。
上面的代码，当我们使用了 `track_caller`,运行代码提示：

```Bash
thread 'main' panicked at examples/track_caller_macro.rs:9:9:
Assertion failed at examples/track_caller_macro.rs:8:22
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

当没有使用 `track_caller`,运行代码提示：

```Bash
thread 'main' panicked at examples/track_caller_macro.rs:2:5:
Assertion failed at examples/track_caller_macro.rs:2:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

这对于编写库代码或者任何需要提供清晰错误信息的函数尤为重要。标准库中的许多函数，如`Option::unwrap`、`Result::expect`和`panic!`宏，都使用了`#[track_caller]`，因此当这些操作失败时，恐慌（panic）信息能够准确地指向用户代码中触发错误的那一行，而不是`unwrap`或`expect`函数本身的实现位置。

## Rust 学习资源

### Flattening Rust's Learning Curve

<https://corrode.dev/blog/flattening-rusts-learning-curve/>

这篇文章对于如何学习Rust提供了非常好的建议，包括

- 不要用其他语言的思维方式来学习Rust，并更好的借助`Clipy lint`检查工具来帮助自己
- 从小的代码开始，逐步引入Rust的相关概念
- 开始阶段不要使用 AI 生成代码，而是手动编写代码
- ....

这篇文章其实不止是学习Rust，对于学习任何一门新的语言和技术都是非常有帮助的。
