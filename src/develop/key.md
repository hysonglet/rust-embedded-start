

# 按键的检测
在本节中将介绍如何使用普通的 GPIO 按键，检测引脚的电平。

## 原理图

由原理图可知：我们的评估版只有两个按键，其中 RST 按键为系统复位按键，BOOT 按键有两个功能，在复位时候按下 BOOT 按键时，单片机将进入串口下载程序模式。当复位时候 BOOT 未按下，则单片机进入用户程序运行的模式，程序运行时，BOOT 按键可作为普通输入检测。

- BOOT 按键的引脚为 PF4
- BOOT 按键释放时候，电平为低电平，按下后电平为高电平
- 建议使用时 PF4 关闭上下拉即可

## 测试代码：`examples/key.rs`
``` rust
#![no_std]
#![no_main]

use {defmt_rtt as _, panic_probe as _};

use embedded_hal::digital::v2::InputPin;
use hal::delay;
use hal::gpio::{Input, PinPullUpDown, PinSpeed};
use py32f030_hal as hal;

#[cortex_m_rt::entry]
fn main() -> ! {
    defmt::println!("examples: key");

    let p = hal::init(Default::default());

    let gpioa = p.GPIOF.split();

    let key = Input::new(gpioa.PF4_BOOT0, PinPullUpDown::No, PinSpeed::Low);

    loop {
        defmt::info!("key: {}", key.is_low());
        delay::delay_ms(1000);
    }
}
```

例程代码非常简单，在循环中每隔 100 毫秒读取一次 引脚电平并打印到终端。

## 编译并烧录
``` bash
cargo r --example key
```

## 测试
```
➜  py32f030-hal git:(main) ✗ cargo r --example key                     
warning: unused manifest key: dependencies.embedded-io-async.option
   Compiling py32f030_hal v0.1.0 (/Users/hunter/mywork/py32/py32f030-hal)
    Finished `dev` profile [optimized + debuginfo] target(s) in 6.42s
     Running `probe-rs run --chip PY32F030x8 target/thumbv6m-none-eabi/debug/examples/key`
      Erasing ✔ [00:00:00] [#################################################] 8.00 KiB/8.00 KiB @ 64.50 KiB/s (eta 0s )
  Programming ✔ [00:00:02] [##################################################] 7.50 KiB/7.50 KiB @ 3.44 KiB/s (eta 0s )    Finished in 2.349s
<lvl> examples: key
└─ key::__cortex_m_rt_main @ examples/key.rs:22  
INFO  key: Ok(true)
└─ key::__cortex_m_rt_main @ examples/key.rs:22  
INFO  key: Ok(true)
└─ key::__cortex_m_rt_main @ examples/key.rs:22  
INFO  key: Ok(true)
└─ key::__cortex_m_rt_main @ examples/key.rs:22  
INFO  key: Ok(false)
└─ key::__cortex_m_rt_main @ examples/key.rs:22  
INFO  key: Ok(false)
└─ key::__cortex_m_rt_main @ examples/key.rs:22  
INFO  key: Ok(false)
└─ key::__cortex_m_rt_main @ examples/key.rs:22  
INFO  key: Ok(true)
└─ key::__cortex_m_rt_main @ examples/key.rs:22  
``` 

## 你可能好奇的地方
1. 为什么设置上拉为：`PinPullUpDown::No`? 
    因为 PF4 引脚内部默认为下拉，确保正常情况单片机能运行主flash程序，如果设置为上拉，可能导致捕获的电平不能识别。取决于开发版的下拉电阻大小。去勇敢尝试 `PullUp/PullDown` 测试一下吧
2. 如果提高检测效率？去开一个 embassy 任务吧，把检测放在单独的任务中，或者使用异步检测方式。