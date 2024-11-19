
# 循环
Rust 也有多种循环语句，能灵活适用于不同的场合。与 C/C++ 区别的是：
1. 循环能有返回值
2. break 可以有返回值
3. 允许迭代器风格的循环
4. `while let` 语句匹配 `Option` 枚举

## 示例
``` rust
fn main() {
    for i in 0..5 {
        println!("{i}");
    }

    let mut cnt = 10;
    let rst = loop {
        cnt -= 1;
        if cnt == 0 {
            break 0;
        }
    };

    while cnt <= 10 {
        cnt += 1;
    }

    let v = vec![1, 2, 3];
    let mut iter = v.iter();
    while let Some(v) = iter.next() {
        println!("{}", v)
    }

    for v in v.iter() {
        println!("{}", *v);
    }
}

```

