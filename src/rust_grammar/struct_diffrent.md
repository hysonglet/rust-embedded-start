
# Rust 的结构体与 C 的结构体的差异
Rust 作为一种系统级编程语言， 与C/C++语言一样，它们都有非常高效的内存读写操作，因此三者都非常适合资源有限的嵌入式平台的开发。
在很多编程语言中都有结构体这个概念，常常用于面向对象开发， 提高代码内聚， 降低耦合。Rust的结构体与C的结构体有非常多的相似之处，甚至可以使用`#[repor(C)]`属性， 直接与C/C++的结构体进行互操作。让 Rust 调用 C 库使用方法和 `C` 与 `C++` 代码互相调用一样简单。但 Rust 的结构题体与 C 的结构体也有非常多的不同之处，有更多的特性，用于简化编程，简化抽象逻辑。

## 定义方法
Rust 结构体的预定义关键字与 `C` 一样，都使用 `struct` 关键字开头。但Rust 定义的后面通常无需添加分号`;`，除非结构体为空。各子属性的后面使用逗号分隔。
Rust 的结构体定义方法如下：
```rust
struct Point {
    x: i32,
    y: i32,
}

// 空结构体定义方法
struct EmptyStruct;
```


## 构造与初始化
Rust 的结构体没有要求必须有构造函数，但可以通过关联函数（如 new）初始化：
```rust
#[derive(Default)]
struct Point {
    x: i32,
    y: i32,
}
impl Point {
    pub fn new(x: i32, y: i32) -> Point {
        Point { x, y }  // 等价于 Point { x: x, y: y }
    }

    pub fn Point(point: Point) -> Point {
        Point {
            x: point.x,
            ..point  // 等价于 Point { x: point.x, y: point.y }
        }
    }
}
```
需要注意的是，Rust 的字段必须都初始化后才能使用，否则编译会报错，来保证字段的安全性。可使用 `#[derive(Default)]` 注解来为结构体添加默认值。

## 内部字段的访问
Rust 的结构体的内部字段可使用可见性修饰符，用于控制字段的访问权限。Rust 的结构体支持以下可见性修饰符：
- ` ` : 默认可见性，字段只能在当前模块内访问。
- `pub`：公共可见性，字段可以被外部访问。
- `pub(crate)`：当前 crate 内可见，字段可以被当前 crate 内的代码访问。
- `pub(super)`：父级模块可见，字段可以被父级模块内的代码访问。

## 内存排列
Rust 的结构体的内存排列与 `C` 结构体的内存排列不同。Rust 的结构体的内存排列是按照字段的定义顺序进行排列的，而 `C` 结构体的内存排列是按照字段的声明顺序进行排列的。
Rust 的结构体的内存排列如下：
```rust
struct Display {
    x: i32,
    width: u8,
    y: i32,
    height: u8,
}
```
以上代码在编译后的内存顺序可能自动调整为：
```rust
struct Display {
    x: i32,       // offset 0, size 4
    y: i32,       // offset 4, size 4
    width: u8,    // offset 8, size 1
    height: u8,   // offset 9, size 1
}
```
如果需要保持字段的内存顺序与C语言一致，可以使用`#[repr(C)]`属性，这样就可以保持字段的内存顺序与C语言一致, 在与C交互时则会完全兼容。 如下：
```rust
#[repr(C)]
struct Display {
    x: i32,       // offset 0, size 4
    width: u8,    // offset 4, size 1
    y: i32,       // offset 8, size 4
    height: u8,   // offset 12, size 1
}
```

## 方法
Rust 的支持为结构体定义方法，方法与函数的定义类似，只是方法定义在结构体的内部。方法的定义格式如下：
```rust
struct Point {
    x: i32,
    y: i32,
}
impl Point {
    fn distance(&self, other: &Point) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        (dx * dx + dy * dy) as f64
    }

    pub fn new(x: i32, y: i32) -> Point {
        Point { x, y }
    }

    pub(crate) fn get_x(&self) -> i32 {
        self.x
    }
    pub(super) fn get_y(&self) -> i32 {
        self.y
    }
}
```
如上所示，使用 `impl` 关键字定义结构体的方法，
- 内部的第一个参数如果为 `self`，表示当前结构体的实例中可使用， 当所有权会被移动。
- 如果方法的第一个参数的类型为 `&self`，表示当前结构体的实例的引用，结构体变量所有权不会移动，无法修改内部属性的值。
- 方法的第一个参数的类型为 `&mut self`，表示当前结构体的实例的可变引用，当前函数可修改内部属性的值。
- 如果函数的第一个参数非 `self`、 `&self`、 `&mut self`，则表示当前函数为静态方法， 可直接通过结构体名调用。

Rust 也能使用 `impl trait_name for struct_name` 来为结构体实现 trait， 如下：
```rust
struct Point {
    x: i32,
    y: i32,
}
impl Display for Point {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "({},{})", self.x, self.y)
    }
}
```
以上代码定义了一个 `Point` 结构体，并为其实现了 `Display` trait， 这样就可以使用 `println!` 宏来打印 `Point` 结构体的实例。
```rust
let p = Point { x: 1, y: 2 };   
// 代码会输出 `(1,2)`。
println!("{}", p);
```

## 所有权与生命周期
Rust 的结构体可以包含引用（&T 或 &mut T），但必须指定生命周期：
```rust
struct Bar<'a> {
    data: &'a str, // 必须标注生命周期
}
```

## 范型化
Rust 的结构体支持模板化， 可以使用泛型来定义结构体， 如下：
```rust
struct Point<T> {
    x: T,
    y: T,
}
```
以上代码定义了一个 `Point` 结构体， 其中 `x` 和 `y` 都是泛型类型 `T`， 可以是任意类型。这样就可以使用 `Point` 结构体来表示任意类型的点, 适应不同的场景。
```rust
let p = Point { x: 1, y: 2 };
let p = Point { x: 1.0, y: 2.0 };
```

## 模式匹配
Rust 的结构体支持模式匹配， 可以使用模式匹配来解构结构体， 如下：
```rust
struct Point {
    x: i32,
    y: i32,
}
fn main() {
    let p = Point { x: 1, y: 2 };
    match p {
        Point { x, y } => println!("x={}, y={}", x, y),
        _ => println!("other"),
    }
}
```
以上代码定义了一个 `Point` 结构体， 并在 `main` 函数中使用模式匹配来解构 `Point` 结构体的实例， 解构后可以使用 `x` 和 `y` 变量来访问结构体的内部属性。

## 析构函数
Rust 的结构体都有析构函数， 析构函数在结构体被释放时自动调用， 可以用于释放资源， 如下：
```rust
struct Foo {
    data: Vec<i32>,
}
impl Drop for Foo {
    fn drop(&mut self) {
        println!("Foo is dropped");
    }
}
```
结构体变量在释放时默认会为内部的字段执行析构函数，避免内存泄漏，保证内存安全。

## 总结
Rust 的结构体与 C 的结构体有非常多的不同之处， 但 Rust 的结构体也有非常多的优点，用于提高内存安全和简化编程， 如：
- 支持范型化， 可以使用泛型来定义不同属性类型的结构体
- 支持模式匹配， 可以使用模式匹配来解构结构体
- 支持析构函数， 可以使用析构函数来释放资源
- 支持生命周期， 可以使用生命周期来指定结构体的引用的生命周期
- 支持引用， 可以使用引用来指向结构体的内部属性
- 支持方法， 可以使用方法来定义结构体的行为
- 支持可见性修饰符， 可以使用可见性修饰符来控制字段的访问权限
- 支持内存排列， 可以使用内存排列来指定结构体的内存布局
- 支持构造函数， 可以使用构造函数来初始化结构体的实例
- 支持静态方法， 可以使用静态方法来定义结构体的行为
- 支持兼容C, 可以使用 `#[repr(C)]` 属性来与C语言交互， 适应多语言编程