
# 结构体
Rust 允许使用结构体自定义类型，与 C 差别如下：
- 可兼容 C 的内存排列
- 默认内部数据可能被编译起自动重新排列
- 允许使用模板
- 可作为对象实现公开或私有的方法
- 可使用属性宏自动实现方法
- 允许空类型
- 无需手动调用释放函数

``` rust
#[repr(C)]
pub struct ExceptionFrame {
    r0: u32,
    r1: u32,
    r2: u32,
    r3: u32,
    r12: u32,
    lr: u32,
    pc: u32,
    xpsr: u32,
}

struc Empty;

impl Empty {
    pub fn say_hello() {
        println!("hello");
    }
}

#[derive(Debug)]
struct Student<'a> {
    name: String,
    age: u8,
    nake: &'a str
}

struct Fruit<T> {
    type: T,
    weight: u8,
}

impl <T> Fruit<T> {
    fn new(t: T, w: u8) -> Self {
        Self {
            type: t,
            weight: w
        }
    }
}

```