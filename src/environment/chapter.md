
# 环境

## Rust 编译环境

 Rust 提供了非常简单的安装命令, 详细可以参考：[Rust 官方入门手册](https://doc.rust-lang.org/book/)
详细请参考： [Installation - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/book/ch01-01-installation.html)
#### Mac/Linux
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
#### [Windows](https://static.rust-lang.org/rustup/dist/i686-pc-windows-gnu/rustup-init.exe)

### 设置 Rust night 版本
Rust 的嵌入式开发环境需要是 nightly 版本
```bash
rustup default nightly
```

## 嵌入式编译工具链

### 安装 Cortex-M0 编译工具
```bash
rustup target add thumbv6m-none-eabi
```
## 仿真环境

## nostd

## 主控

## 开发版