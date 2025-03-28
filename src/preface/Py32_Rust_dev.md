
## PY32F030_Rust_Dev

`PY32F030_Rust_Dev` 的主要目的是提供一个最简单的嵌入式 Rust 入门开发板，主控选择国产芯片普冉单片机 PY32F32030，内核为 Cortex-M0+，价格便宜、资源丰富。该款单片机在很多产品中有所覆盖，如低功耗智能门锁、吹风机、电子烟、玩具等。目前已适配了大部分的外设如GPIO、I2C、Timer、RTC、Flash、Exti等外设。驱动分层明确，提供了大量的注释，便于深入理解嵌入式Rust驱动的设计模式。

![顶部](./img/board_top.png)
![底部](./img/board_botton.png)

除支持 Rust 开开发外，对于传统的C/C++或Arduno框架，官方和第三方社区也有丰富的开源代码可以在本开发板上运行。对于习惯传统C/C++开发的同学，选择官方驱动库，在 Keil、IAR、EIDE 上开发，也非常成熟， 有大量的资料可以深入学习。

## 软件资源

## OpenPuraOpenPuya (github.com)
Arduno
py32duino/Arduino-PY32: Arduino core for the PY32. (github.com)
Cmake
decaday/py32f0-cmake: PY32F0 MCU CMake project template. Using GCC, cross-platform.Supported PY32F030,PY32F002/3,XL32,AIR001 (github.com)
IAR/Keil/GCC/EIDE

EIDE
1. 设置工具链环境
MAC下
1. 下载 .NET 9.0 (Linux、macOS 和 Windows) (microsoft.com)
Mac(M1) 安装.net 教程_mac 安装dotnet-CSDN博客
2. brew install openocd

Rust 
1. decaday/py32-bind-hal: Py32(Puya) Series MCU HALs using binding (github.com)
2. 