
# 常用工具和命令

## Cargo

### Cargo
- [Cargo](./cargo.md)
- [svd2rust](./svd2rust.md)
- [串口工具](./serial_tools.md)


## GDB

用来调试单片机，查看或修改芯片内部数据
[GDB 调试 Rust 嵌入式程序-以国产单片机为例](https://mp.weixin.qq.com/s/xPEa6lbgIghFrpBeDgdxSw)
TODO
``` bash
ext 3333
```

[Overview of embedded programming and debugging - Debugonomicon (rust-embedded.org)](https://docs.rust-embedded.org/debugonomicon/)
## Probe-rs
Probe-rs 用于下载和调试固件，通常可使用 Jlink、Stlink、DapLink等工具下载。
目前 `Py32 Rust Dev` 可使用 Jlink/Stlink 下载固件并打印 `defmt` 或 `RTT` 日志。在 demo 中直接使用`cargo r`等命令即可。

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
