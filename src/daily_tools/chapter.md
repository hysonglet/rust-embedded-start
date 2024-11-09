
# 常用工具和命令

## Cargo

### 检查
只检查 crate 语法，不编译，通常用来节省时间
```
cargo check
# or
cargo c
```

### 编译
编译 crate 或 指定的例程目标
``` bash
# 编译当前 crate
cargo build
# or
cargo b
# 编译 example 目录下的例程文件 XX
cargo b --example XX
# 以 release 模式编译例程 XX
cargo b --example XX --release
# 编译例程并使能 feature FFF
cargo b --example XX --feature FFF
```

### 下载 & 运行
下载 elf 固件到芯片并运行，probe-rs 会实时打印来自单片机的日志
``` bash
# 下载并运行
cargo run
# or
cargo r
# 以 release 模式运行
cargo r --release
# 运行 例程
cargo r --example blinky -release
```

## GDB
用来调试单片机，查看或修改芯片内部数据
TODO
``` bash
ext 3333
```

## Probe-rs
### 查看支持的芯片清单

### 烧录

### 重启设备

### 临时探测日志
TODO
``` bash
probe-rs attach 
```

### 指定 jlink

### 读写内存或寄存器

## 串口烧录


## 附录
- [The Cargo Book](https://doc.rust-lang.org/cargo/index.html)

- [About probe-rs](https://probe.rs/docs/overview/about-probe-rs/)

- [GDB Tutorial](https://www.cs.umd.edu/~srhuang/teaching/cmsc212/gdb-tutorial-handout.pdf)