---
title: tips-for-faster-rust-compile
date: 2025-07-31 08:57:00
tags:
  - rust
categories:
  - rust
published: false
---
## Updatge Rust Compiler And ToolChain

Rust 一直在持续不断的优化编译的速度，因此定期更新你的 Rust 编译器和工具链是非常重要的。

```Rust
rustup update
```

## Cargo Check

Cargo check 是一个非常有用的工具，它可以在不编译代码的情况下检查你的代码是否有语法错误或其他问题。这可以大大加快你的开发速度。

## Remove Unused Dependencies

```Rust
cargo install cargo-machete && cargo machete
```

## Update Dependencies

- `cargo update` 会自动更新所有依赖项到最新版本。
- `cargo tree --duplicate` 查找多个版本的依赖项，目的是通过更新依赖旧版本的依赖项来合并到单个版本(待验证)

## Faster Linker

- Linux/Macos: <https://lld.llvm.org/>
- Linux: <https://github.com/rui314/mold>

目前github 上比较活跃的还有： <https://github.com/davidlattimore/wild> 可以根据实际的情况选择

## MacOS: Faster Incremental Debug Builds

在 `Cargo.toml` 中添加

```Toml
[profile.dev]
split-debuginfo = "unpacked"
```
