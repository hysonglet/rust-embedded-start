
# 深入理解 hello world 例程

## 工程解说
`py32f030-hal` crate 为 py32f030 芯片的外设支持库, 提供基本的外设访问接口。

`py32f030-hal` crate 的目录结构如下：
``` text
➜  py32f030-hal git:(main) ✗ tree -L 1
.
├── Cargo.lock
├── Cargo.toml
├── Embed.toml
├── LICENSE
├── README.md
├── examples
├── memory.x
└── src
```
初次学习时，该目录中有以下几个目录或文件需要了解：
### Cargo.toml
该文件为 crate 的包管理文件，通过 toml 文件格式设置 crate 的基本信息，常用属性如下
- crate 版本
- 包名
- 依赖
- features
- 编译属性

 Rust 编译的最小单位为 crate，发布的单位也为 crate。通常 crate 分为两种，
 - 二进制 crate：直接编译成可执行的二进制，将包含一个 main 函数入口
 - 库 crate，提供库接口，编译后为 lib 文件。也可添加多个 example 用来提供接口使用样例，可在 example 文件中包含 main 函数用于直接生成可执行的测试固件。

 详细可参考：[TOML 格式详解](../daily_tools/toml_format.md)

``` toml
[package]
name = "py32f030_hal"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
[dependencies]
bare-metal = { version = "1.0.0" }
cast = "0.3"
cortex-m-rt = "0.7"
critical-section = { version = "1.1.2" }
embedded-hal = { version = "0.2", features = ["unproven"] }
embedded-hal-async = { version = "1.0" }
...
[features]
default = ["example"] # "embassy"
example = ["dep:defmt", "dep:defmt-rtt", "dep:panic-probe", "dep:panic-halt"]
embassy = [
    "dep:embassy-executor",
    "dep:embassy-sync",
    "dep:embassy-time",
    "time-driver",
]
## Enable the timer for use with `embassy-time` with a 1KHz tick rate.
time-driver = ["dep:embassy-time-driver", "embassy-time-driver/tick-hz-1_000"]

[dev-dependencies]

[[example]]
name = "blinky"

[profile.dev]
codegen-units = 1
opt-level = "z"
lto = "fat"
debug = true
overflow-checks = true
strip = false

[profile.release]
codegen-units = 1
opt-level = "z"
lto = true
debug = false
overflow-checks = false
# strip = true   # 开启后release模式的日志不会显示
```

### Embed.toml

`Embed.toml` 是 cargo 插件 `cargo-flash` 工作依赖的文件，通常来说并不是 Rust crate 中所必备的文件，在嵌入式开发中通常可以用来使用插件命令 `cargo embed <args>
` 来快速编译、运行、或调试固件。详细可了解：[cargo-embed](https://probe.rs/docs/tools/cargo-embed/)
``` toml
➜  py32f030-hal git:(main) ✗ cat Embed.toml 

[default.general]
chip = "PY32F030x8"

[default.rtt]
enabled = false

[default.gdb]
enabled = true

[default.reset]
halt_afterwards = true
```

在以上代码中则指定命令 `cargo embed` 执行时所依赖的芯片名， 重启后进入 GDB 状态。

### Examples
该目录用来存放一些测试样例代码，如 blinky，uart 等，初学者可以通过运行这些测试代码来快速了解如何使用外设驱动，crate 根目录下直接执行： `cargo r --example expample_file_name`

``` text
➜  py32f030-hal git:(main) ✗ tree -L 1 examples 
examples
├── adc_block.rs
├── advanced_timer_block.rs
...
├── hello_world.rs
├── i2c_master_block.rs
├── key.rs
├── rtc_block.rs
└── uart.rs
```
### memory.x

`memory.x` 文件用来指定芯片 RAM 和 Flash 等存储器的大小和地址，类似 C 中的链接脚本 `ld` 文件。在本 crate 中非必需，目前仅当使用 example 时才会用到该文件。在自己新建的 bin crate 工程中需要包含芯片对应 memory.x 文件。

```
MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 64K
  RAM : ORIGIN = 0x20000000, LENGTH = 8K
}
```
### src

`src` 目录为该库的源文件，通常在单片机外设库中，每个外设作为一个单独的模块。模块的根节点在 lib.rs 中，模块名即为包名，也就是 `py32f030_hal`。
```
➜  py32f030-hal git:(main) ✗ tree -L 1 src     
src
├── lib.rs
├── adc
...
├── rtc
├── spi
├── syscfg
├── timer
└── usart
```

## main 函数
对于大多数嵌入式工程来说，因 flash 和 ram 资源有限，无法容纳标准 `std` 接口的代码 ，因此需要指定 rust 引用 非标准库(no_std)。

hello-world.rs 代码如下：
``` rust
#![no_std]
#![no_main]

use py32f030_hal as _;

// use panic_halt as _;
use {defmt_rtt as _, panic_probe as _};

#[cortex_m_rt::entry]
fn main() -> ! {
    defmt::info!("hello world");
    loop {
        cortex_m::asm::wfe();
    }
}
```