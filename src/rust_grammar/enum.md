
# 枚举

Rust 的枚举可与C兼容，也有升级之处。
- Rust 枚举项可包含其他类型
- Rust 枚举可使用模板
- Rust 枚举的大小与枚举类型有关，并非一定为整型

``` rust
enum Ea{
    A0,
    A1,
}

pub enum Eb{
    B0 = 0,
    B1 = 1,
}

pub(crate) enum Ident{
    ID(u32),
    Num(i32),
    Key(String),
}

enum Rst<T, E> {
    OK(T),
    Err(E),
}
```

## `Resoult<T, E>` 和 `Option<T>`
`Resoult<T, E>` 和 `Option<T>` 这两个枚举由核心库提供，在 Rust 编程中经常使用。原型如下：
``` rust 
pub enum Option<T> {
    /// No value.
    None,
    /// Some value of type `T`.
    Some(T),
}

pub enum Result<T, E> {
    /// Contains the success value
    Ok(T),

    /// Contains the error value
    Err(E),
}
```

`Option<T>` 经常用于表示有无，`Result<T, E>` 常用于返回结果。