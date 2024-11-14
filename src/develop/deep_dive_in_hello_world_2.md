# 深入理解 hello world 例程——基于 cortex-m-quickstart

在本篇中，将带你手把手新建一个基于常用的单片机主板的 hello world 工程。

## 工具链准备

如果您还没有安装 Rust 的基本开发环境，您可参考 [快速搭建环境](../environment/chapter.md)。

首先，你需要了解你手上的单片机的处理器内核，如 STM32F103 为 Arm® Cortex®-M3 CPU，NRF52840 为 Arm® Cortex®-M4F CPU，CH32V307 为 RISC-V4F 内核。因此你需要根据你芯片内核的来安装相应的交叉编译链。 常见的嵌入式交叉编译工具如下。

``` bash
➜  rust-embedded-start git:(main) ✗ rustup target list | grep none
aarch64-unknown-none
aarch64-unknown-none-softfloat
armebv7r-none-eabi
armebv7r-none-eabihf
armv7a-none-eabi
armv7r-none-eabi
armv7r-none-eabihf
loongarch64-unknown-none
loongarch64-unknown-none-softfloat
riscv32i-unknown-none-elf
riscv32im-unknown-none-elf
riscv32imac-unknown-none-elf
riscv32imafc-unknown-none-elf
riscv32imc-unknown-none-elf
riscv64gc-unknown-none-elf
riscv64imac-unknown-none-elf
thumbv6m-none-eabi (installed)
thumbv7em-none-eabi
thumbv7em-none-eabihf
thumbv7m-none-eabi (installed)
thumbv8m.base-none-eabi
thumbv8m.main-none-eabi
thumbv8m.main-none-eabihf
x86_64-unknown-none
```

Rust 交叉工具链的目标是由三元组组成，也叫目标三元组。

它通常采用 `<arch>-<vendor>-<sys>-<abi>` 的形式来描述目标平台的架构、供应商、操作系统和应用程序二进制接口（ABI）。
- **架构（arch）**
    - 例如，`thumbv7m - none - eabi`中的`thumbv7m`表示 ARM 架构下的 Thumb - 2 指令集的 v7m 版本。这告诉编译器生成的代码要符合这种特定的 ARM 架构指令集要求，不同的 ARM 指令集（如 ARMv8、Thumb - 2 等）有不同的性能和功能特点，编译器需要根据这个信息生成合适的机器码。
    - 对于`powerpc - unknown - linux - gnu`中的`powerpc`，它代表 PowerPC 架构，这是一种与 ARM 不同的处理器架构，具有自己的指令集和硬件特性，编译器要按照 PowerPC 的规则来生成代码。
- **供应商（vendor）**
    - 在目标三元组中，供应商部分可以提供关于硬件制造商的信息。不过在很多情况下，`none`这样的标识被使用，表示这不是特定某个供应商的硬件定义，或者是通用的定义。例如，在`thumbv7m - none - eabi`中，`none`表示这个定义不是针对某一个特定的 ARM 芯片供应商（如三星、恩智浦等），而是一种通用的 ARM Thumb - 2 v7m 指令集的定义。
- **操作系统（sys）**
    - 操作系统部分明确了目标代码运行的操作系统环境。例如，`linux`表示目标代码是运行在 Linux 操作系统之上。这会影响编译器如何处理系统调用、库链接等操作。对于嵌入式系统，可能会看到`none`（表示没有操作系统，如裸机环境），像`thumbv7m - none - eabi`中的`none`就表明这个代码可能是用于没有操作系统的 ARM 嵌入式设备，编译器就不会生成与复杂操作系统交互的代码部分。
    - 而`powerpc - unknown - linux - gnu`中的`linux`说明代码是为运行在 PowerPC 架构的 Linux 系统准备的，编译器需要确保生成的代码能够与 Linux 的系统调用接口、文件系统等兼容。
- **应用程序二进制接口（ABI）**
    - ABI 部分定义了二进制层面上函数调用、数据结构布局等的规则。例如，`eabi`（嵌入式应用程序二进制接口）在`thumbv7m - none - eabi`中是用于嵌入式系统的 ABI 标准。它规定了如何在二进制层面上传递参数、返回值等。不同的 ABI 标准适用于不同的应用场景，`gnu`（如在`powerpc - unknown - linux - gnu`中）是一种在 Linux 系统上常用的 ABI，遵循 GNU 的规则来处理函数调用和数据结构布局等。

在清楚对应的交叉编译起三元组名后，你需要使用命令安装它，如对于 Py32f030 芯片来说，内核是 `ARM Cortex-M0+` , 无浮点加速，因此需要安装 `thumbv6m-none-eabi` 

```
rustup target add thumbv6m-none-eabi
```
其他常见 内核 的单片机与交叉编译器对应如下：
- ARM Cortex-M0: `thumbv6m-none-eabi`
- ARM Cortex-M3： `thumbv7m-none-eabi`
- ARM Cortex-M4: `thumbv7em-none-eabi`
- ARM Cortex-M7:  `thumbv7em-none-eabi`
- ARM Cortex-M33: `thumbv8m.main-none-eabi`
- ARM Cortex-M4**F**: `thumbv7em-none-eabihf`
- ARM Cortex-M7**F**: `thumbv7em-none-eabihf`

如果使用错误的交叉编译器编译 Rust 代码，可能编译失败或生成的固件运行指令会异常。

## 创建工程

基于 模板创建新工程。

``` bash
git clone cortex-m-quickstart --vers 0.3.0
```
