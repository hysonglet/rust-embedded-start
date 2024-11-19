
# 匹配
`match` 模式匹配 与 C/C++ 有点类似，都用于匹配多个值。与 C/C++ 不同如下：
1. match 语句最后会有返回值
2. 每个分支的返回值类型必须一致
3. 支持模糊匹配
4. 无 break，不会出现贯穿错误
5. 必须匹配所有范围，否则编译报错, 通常使用 `_` 替代其他值。

## 实例
```rust
#[derive(PartialEq)]
enum EType {
    Ta,
    Tb,
    Tc(i32),
}

fn main() {
    let e1 = EType::Tc(10);

    match e1 {
        EType::Ta => {
            println!("ta");
        }
        EType::Tb => {
            println!("tb");
        }
        EType::Tc(v) => match v {
            0 => println!("tc: 0"),
            _ => println!("tc: {v}"),
        },
    }

    let e2 = EType::Tb;

    let rst = match e2 {
        EType::Ta | EType::Tb => 0,
        EType::Tc(v) => v,
    };
    println!("rst: {rst}");

    let cnt = 10;
    match cnt {
        0..6 => {
            println!("0~5")
        }
        6..10 => {
            println!("6~10")
        }
        _ => {
            println!("others")
        }
    }
}
```