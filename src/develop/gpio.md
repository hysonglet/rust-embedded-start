
# GPIO 

GPIO(General Purpose Input/Output), 代表通用输入/输出引脚，在微控制器的大部分引脚都可用于GPIO用途。

GPIO 引脚可配置为输入或输出功能，也可复用为其他外设的功能引脚。作为输入引脚时，单片机内部可读取引脚的外部电平，当作为输出引脚时，单片机可以配置寄存器来输出高/低电平。

## 如何控制寄存器
通过配置 GPIO 外设寄存器，将GPIO引脚配置成相应的功能。对于普通输入或输出引脚的来说，通常需要关注以下配置：
- 上拉/下拉
- 开漏/浮空模式
- 速度
- IO模式

当前外设支持库的 GPIO mod 提供了基本的API供使用:
``` rust
//! General purpose input/output (GPIO) driver.
//!
//! Output
//!
//! ```rust, ignore
//! let p = hal::init(Default::default());
//! let gpioa = p.GPIOA.split();
//! let mut led = Output::new(gpioa.PA10, PinIoType::PullDown, PinSpeed::Low);
//! ```
//!
//! Input
//! ```rust, ignore
//! let p = hal::init(Default::default());
//! let gpioa = p.GPIOA.split();
//! let key = Input::new(gpioa.PA12, PinPullUpDown::PullUp, PinSpeed::Low);
//! ```
//!
//! AF
//! ```rust, ignore
//! let gpioa = p.GPIOA.split();
//! let _mco_pin = Af::new(
//!    gpioa.PA1,
//!    PinAF::AF15,
//!    PinSpeed::VeryHigh,
//!    PinIoType::PullUp,
//! );
//! Mco::select(clock::McoSelect::SysClk, clock::McoDIV::DIV1);
//! ```
```