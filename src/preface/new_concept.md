# C 嵌入式转 Rust 嵌入式需要面对的新概念

## 模块

## Crate
`crate` 是 Rust 的代码单元，也是 Cargo 管理、发布的最小单元。通常可以包含一个或多个模块。Rust有两种 crate ：库（Library Crate）和二进制（Binary Crate）。
- `Library Crate` 用于提供代码库，编译后的目标文件是链接库，即动态库、静态库等，可用于Rust或第三方语言（如C/C++等）调用。当然Libary crate也可以被其他crate引用，在Cargo.tml 中指定引用信息，从而直接使用改crate的模块或 API 。大致可以理解为 C/C++ 的库，例如 stdio.h,string.h等。 
- `Binary Crate` 用于实现具体的程序，包含程序入口 main 函数，编译后为二进制的程序，可直接运行。可以简单理解为 C/C++ 的 `main.c + XX.c`文件，当然实际用起来比它们更加优雅。

## Package
`Package` 用于管理 Crate, 每个 package 的根目录包含一个文件 `Cargo.toml` ,用于指定 Package 的元信息，如作者、版本、features、依赖、目标编译平台。Cargo 可将 Package 发布到 crate.io。
一个 Package 可以包含多个字 成员（members）

## Cargo

## features

[Introduction - The Cargo Book (rust-lang.org)](https://doc.rust-lang.org/cargo/index.html)

## 枚举
- Result/Option

## #![no_std]
## 异步/Future
## 零成本抽象
