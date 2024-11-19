
# 在不同的任务中点灯

在上一篇中，我们已经知道如何点亮一个 LED 灯，那么在这一篇，我们即将提升难度，在不同的任务中点灯，感受并行编程的快乐。

请在你感到好奇或疑惑的地方修改代码！

## 代码： `examples/embassy_blinky.rs`
``` rust
//! 在不同的任务中闪烁不同的 LED， 学习多任务操作
//!

#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::Timer;
use embedded_hal::digital::v2::ToggleableOutputPin;
use hal::gpio::{Output, PinIoType, PinSpeed};
use py32f030_hal::{
    self as hal,
    gpio::{AnyPin, Pin},
};

use {defmt_rtt as _, panic_probe as _};

#[embassy_executor::task(pool_size = 2)]
async fn run_led(led: AnyPin, delay_ms: u64) {
    let mut led = Output::new(led, PinIoType::PullDown, PinSpeed::Low);
    loop {
        let _ = led.toggle();
        Timer::after_millis(delay_ms).await;
    }
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = hal::init(Default::default());

    defmt::info!("Testing the flashing of different LEDs in multi-tasking.");

    let gpioa = p.GPIOA.split();

    // spawner.must_spawn(run_led(gpioa.PA9.degrade(), 1000));
    // spawner.must_spawn(run_led(gpioa.PA10.degrade(), 2000));

    spawner.spawn(run_led_1(gpioa.PA9.degrade(), 1000)).unwrap();
    spawner.spawn(run_led_1(gpioa.PA10.degrade(), 500)).unwrap();

    loop {
        Timer::after_secs(2).await;
    }
}
```

图片：TODO

### 也许你有疑惑的地方

1. `pool_size = 2`  表示异步任务 `async fn run_led(led: AnyPin, delay_ms: u64)` 最多能被 2 个任务使用
2. `async` 表示函数为异步函数
3. `embassy_executor::task` 表示将定义一个异步任务
4. `Spawner` 是异步执行器框架提供的一个类型，它的主要职责是在执行器的任务调度环境中创建新的异步任务。它封装了任务创建的具体细节和逻辑，例如确定任务应该在哪个任务池中运行、分配任务所需的资源（如栈空间等）以及将任务注册到执行器的任务队列中以便后续调度执行。
5.  embassy 的 异步入口函数定义为：
``` rust
#[embassy_executor::main] 
async fn main(spawner: Spawner) {

}
```
6. `spawner.spawn()`: `spawn` 方法是将一个异步函数（或异步闭包）包装成一个任务，并将其放入异步执行器的任务调度队列中，以便在合适的时候执行。这个方法是异步执行器框架提供的关键接口，用于启动新的异步任务并将其纳入任务管理体系。
7. `let _ = led.toggle();` 用于避免编译器报警，忽视 `led.toggle()` 的返回结果
8. 在例程中实际会有 3 个任务运行
