# C 转 Rust 嵌入式需要了解的新概念

## 模块

与 C++的命名空间和Pyhton的模块不同，Rust 的模块更加灵活且作用清晰。
Rust 的模块（mod）用于封装一个逻辑块，mod 可以嵌套定义，设计成层次逻辑，且可以指定 mod 的访问属性，即私有（private）和公开（pub），两者又可细分为：
- `private`: 该模块内部仅模块内部可见
- `pub(crate)`: 模块内部对该 crate 可见
- `pub(super)`：模块内部仅对本级和上级 mod 可见
- `pub`：该模块对全局可见，如果该模块被上级包含，则由上级模块最终决定最大可见范围
通常一个文件名自动成为模块名。对于文件夹来说，模块的根节点为`mod.rs`,模块名则为文件夹的名字

## Crate

`crate` 是 Rust 的代码单元，也是 Cargo 管理、发布的最小单元。通常可以包含一个或多个模块。Rust有两种 crate ：库（Library Crate）和二进制（Binary Crate）。
- `Library Crate` 用于提供代码库，编译后的目标文件是链接库，即动态库、静态库等，可用于Rust或第三方语言（如C/C++等）调用。当然Libary crate也可以被其他crate引用，在Cargo.tml 中指定引用信息，从而直接使用改crate的模块或 API 。大致可以理解为 C/C++ 的库，例如 stdio.h,string.h等。 
- `Binary Crate` 用于实现具体的程序，包含程序入口 main 函数，编译后为二进制的程序，可直接运行。可以简单理解为 C/C++ 的 `main.c + XX.c`文件，当然实际用起来比它们更加优雅。

## Package

`Package` 用于管理 Crate, 每个 package 的根目录包含一个文件 `Cargo.toml` ,用于指定 Package 的元信息，如作者、版本、features、依赖、目标编译平台。Cargo 可将 Package 发布到 crate.io。
一个 Package 可以包含多个成员（members）

## Cargo

Cargo 是 Rust 的包管理工具和构建系统。它在 Rust 开发中起着至关重要的作用，主要有以下功能特点：
- 提供快捷的工程创建、编译、测试、程序下载、运行、文档生成等命令
- 版本控制、依赖控制功能
- 工程管理、发布功能
- 统一工程结构，便于理解和维护，提高开发效率和代码质量。

其功能远比 传统的 Makefile 或 CMake 丰富，但使用却更加简单高效。

## features

features 在 Cargo.tml 中定义，类似 C/C++ 中编译宏的作用，可以选择性编译开启指定 feature 的代码段，从而达到缩小代码体积、实现不同的功能、满足不同的平台等功能。

## 枚举

Rust 的枚举与 C/C++ 的兼容，但表现形式更加丰富。不仅允许指定数字作为枚举值，同时也允许枚举项能附带一个其他类型的对象，如常用的基本类型、其他枚举、结构体、元组等类型，使用枚举能更加轻松直接的描述对象特征和逻辑。使得业务逻辑更加简单清晰。

```rust
pub enum {
    Monday,
    Tuesday,
    ...
}

enum Fruit{
    Apple = 0,
    Orange = 1
}

pub enum {
    Foo,           
    Fruit(Fruit),      /* 附带另外的枚举值 */
    Id(u32),        /* 附带一个 u32 的数字 */
    Human((u8, String)),   /* 附带一个元组 */
    Stu(Student),        /* 附带一个结构体 */
}

```
Rust 有两个特别重要的枚举，经常在代码中可以看到，分别是：
- `Result`：Reuslt 常用于表示结果，定义如下：
    ```rust
    enum Result<T, E> {
        Ok(T),
        Err(E),
    }
    ```
    T 和 E 分别表示正常和异常返回的类型。该定义用于表示正常返回和异常情况返回的结果，各自附带一个类型用于返回有用的信息。该设计能避免像 C 中常使用的数字或宏返回错误值。
- `Option`： 用于表示有无，定义如下：
    ```rust
    enum Option<T> {
        None,
        Some(T),
    }
    ```
    T 表示数据存在时候的数据类型。使用 Option 可以避免使用 null 指针之类不安全的方式来表示有无返回对象。能提高代码可靠性、同时拥有高的执行效率。

## #![no_std]

`no_std` 环境区别于 `std`, `no_std`通常用于嵌入式裸机环境, 降低标准库所占用的程序空间。

no_std 通常不包含操作系统的基本 api，如文件系统、网络、多线程等接口。但其仍然提供了一些常用的基本接口模块，如基本数据类型，算术运算，位操作、裸地址读写等常用接口。在嵌入式裸机系统工程搭建时，需要在主模块中添加 `#![no_std]` 来告诉编译起不要链接 `std` crate。同时对于`no_std` 工程来说，需要自己实现 `panic` 接口。

## Traits

Rust 是一门函数式编程的语言，`trait` 在该方案中发挥了巨大的作用。`trait`可翻译为特征，用于描述对象所具备的 api 能力或属性，也可作为标记，用于提示编译器在编译代码时确定逻辑是否合法，且没有任何运行时的消耗，也不占用程序空间。

不同于面向对象编程中的继承的思想，`trait` 主要用来定义共享的行为，可以在模板中作为一种约束，避免模块实例化有不符合该 trait 约束的对象引入，保证在编译能发现非法的模板使用。Rust 保证在实现灵活的编程时，又能安全、高效的定义接口，避免用户使用错误，同时也能统一接口，极大得方便移植，在常见的 Rust 嵌入式驱动中可以经常看到 `trait` 的使用。

另外常见的标记类的 `trait` 有 `Send`, `Sync`, `Unpin`，`Clone`等

## 异步/Future

在 Rust 中，实现了 `Future` trait 的对象代表可能尚未完成计算的值。Future 有一个关联的 Output 类型，它最终会解析为这个类型的值。Future trait 的定义特征是 poll 函数。

简单的理解可以认为：Future 表示一个异步操作的潜在结果。它允许你以一种非阻塞的方式执行异步任务，并在任务完成时获取结果。不像 C 中常使用注册的 `callback` 函数那样需要程序员清晰得去理解多任务间状态逻辑。Future 让 Rust 异步程序逻辑更加线性化，方便编写和理解。
以下展示一个简单的 异步示例：
```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct MyFuture {
    // 一些状态，例如异步操作的计数器
    counter: u32,
}

impl Future for MyFuture {
    type Output = u32;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.get_mut().counter < 5 {
            self.get_mut().counter += 1;
            Poll::Pending
        } else {
            Poll::Ready(self.get_mut().counter)
        }
    }
}
```
在这个示例中，MyFuture 是一个自定义的 Future，它在计数器达到 5 之前一直处于 Pending 状态，当计数器达到 5 时，返回 Poll::Ready(counter)。
总之，Future trait 和 poll 函数是 Rust 异步编程的基础，它们提供了一种强大的方式来处理异步操作和构建高效的异步程序。

## 零成本抽象

零成本抽象也是 Rust 的一个重要的特点。通常在高级语言中，对于抽象级别越高的对象，可能需要消耗更多的程序空间或运行时间。但在 Rust 中，很多高级抽象都是零成本的，不占用任何运行时间和程序空间。常见如下

- 迭代器
- 闭包
- 智能指针
- 枚举
- 零大小结构体

这些常用的抽象可以放心在嵌入式中使用，不会占用大量的空间，且执行效率与直接手写的代码几乎一致。


---

## 附录

[Introduction - The Cargo Book (rust-lang.org)](https://doc.rust-lang.org/cargo/index.html)

[The Rust Programming Language](https://doc.rust-lang.org/nightly/book/title-page.html)

[The Rustonomicon](https://doc.rust-lang.org/nomicon/)