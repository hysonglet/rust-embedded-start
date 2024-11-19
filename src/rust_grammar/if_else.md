
# 流程控制

Rust 也使用 `if-else`,`if-else if-else` 语句做为流程控制。与 C/C++ 有以下差异：
- 关键字后面不需带括号`()`
- 无论流程内部执行多少行代码，都必须添加花括号`{}`
- `if-else`,`if-else if-else`等语句能返回值，且不同分支的返回类型必须一致，如果不需要返回值，则默认返回为空类型`()`。返回的值为每个分支的最后一个值。
- 判断的表达式的值只能为布尔值，整型值等其他都非法
- `if let` 语句用于 Option 枚举

## 例
``` rust
#[derive(PartialEq)]
enum EA {
    AA,
    BB,
}

fn main() {
    if true {
        println!("true");
    }

    let v = 100;
    if v > 0 {
        println!("true")
    }

    let a = 1;
    let b = 2;

    let max = if a > b { a } else { b };
    println!("{}", max);

    let c = 3;

    let rst = if c < 0 {
        -c
    } else if c == 0 {
        0
    } else {
        c
    };
    println!("{rst}");

    let ea = EA::AA;

    if ea == EA::AA {
        println!("hello");
    }

    if let Some(v) = None {
        println!("{}", v);
    }
}
```
