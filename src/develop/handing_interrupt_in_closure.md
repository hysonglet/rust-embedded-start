# 在单片机中使用闭包优雅应对中断

## 什么是闭包
闭包这个词语由来已久，自上世纪 60 年代就由 Scheme 语言引进之后，被广泛用于函数式编程语言中，进入 21 世纪后，各种现代化的编程语言也都不约而同地把闭包作为核心特性纳入到语言设计中来。那么到底何为闭包？

闭包是一种匿名函数，它可以赋值给变量也可以作为参数传递给其它函数，不同于函数的是，它允许捕获调用者作用域中的值，例如：

``` rust
fn main() {
   let x = 1;
   let sum = |y| x + y;

    assert_eq!(3, sum(2));
}
```
上面的代码展示了非常简单的闭包 `sum`，它拥有一个入参 y，同时捕获了作用域中的 x 的值，因此调用 sum(2) 意味着将 2（参数 y）跟 1（x）进行相加，最终返回它们的和：3。

可以看到 `sum` 非常符合闭包的定义：可以赋值给变量，允许捕获调用者作用域中的值。

# 使用闭包优雅处理中断
本篇文章将介绍如何使用闭包优雅处理中断。Rust 使用闭包处理中断的设计模式，目的是让中断处理更加及时、同时让 Rust 中的所有权检查能够顺利通过，也让API的使用更加简洁，符合人体工学设计，同时保证上层应用与底层逻辑的分离。

## 代码示例：`examples/adc_block_interrupt_closure.rs`
``` rust
#![no_std]
#![no_main]

extern crate alloc;

use defmt::Debug2Format;
use hal::adc::{AdcChannel, AnyAdc, ChannelConfig, Config, Event, SampleCycles, TrigleSignal};

use py32f030_hal::adc::ConversionMode;
use py32f030_hal::clock::peripheral::PeripheralInterrupt;
use py32f030_hal::clock::sys_core_clock;
use py32f030_hal::{self as hal, mode::Blocking};

use {defmt_rtt as _, panic_probe as _};

#[cortex_m_rt::entry]
fn main() -> ! {
    // -------- Setup Allocator --------
    const HEAP_SIZE: usize = 128;
    static mut HEAP: [u8; HEAP_SIZE] = [0; HEAP_SIZE];
    #[global_allocator]
    static ALLOCATOR: alloc_cortex_m::CortexMHeap = alloc_cortex_m::CortexMHeap::empty();
    unsafe {
        ALLOCATOR.init(
            &mut HEAP as *const u8 as usize,
            core::mem::size_of_val(&HEAP),
        )
    }
    let p = hal::init(Default::default());

    defmt::info!("{}", sys_core_clock());

    let mut adc: AnyAdc<_, Blocking> = AnyAdc::new(
        p.ADC,
        Config::default().sample(SampleCycles::Cycle_239_5),
        ChannelConfig::default()
            .over_write(false)
            .wait(true) // 转换完成后等待读取完毕再开始转换
            .singal(TrigleSignal::Soft)
            .mode(ConversionMode::Continuous),
        &[AdcChannel::Channel11, AdcChannel::Channel12],
        // &[AdcChannel::Channel11],
    )
    .unwrap();

    // 使用闭包的方式在中断中调用闭包处理函数
    // 兼顾友好型 api
    static mut QUEUE: [u16; 16] = [0; 16];
    adc.on_interrupt(
        Event::EOC.into(), /* EOC 中断 */
        alloc::boxed::Box::new(move |adc| {
            /* 中断自动调用的闭包 */
            static mut CNT: usize = 0;
            unsafe {
                QUEUE[CNT] = adc;
                CNT += 1;
                if QUEUE.len() == CNT {
                    CNT = 0;
                }
            }

            // 打印转换成功的adc, 打印耗时会导致打印完毕后直接再次进入中断哦，
            // 导致很难看到loop里面的打印
            // defmt::info!("adc: {}", adc);
        }),
    );

    // 开启 EOC 中断
    adc.event_config(Event::EOC, true);
    adc.id().enable_interrupt();
    adc.start();
    loop {
        cortex_m::asm::wfi();

        defmt::info!(
            "adc {:?} sum: {} avrage: {}",
            Debug2Format(unsafe { &QUEUE }),
            unsafe { QUEUE.iter().sum::<u16>() },
            unsafe { QUEUE.iter().sum::<u16>() / QUEUE.len() as u16 }
        );
    }
}
```

测试代码中使用接口来注入闭包函数，让上下文能够轻松得进入中断处理函数中，中断发生时能被调用。
``` rust
pub fn on_interrupt(
        &mut self,
        events: EnumSet<Event>,
        callback: alloc::boxed::Box<dyn Fn(u16)>,
) 
```

那么如何实现这种设计模式呢? 在ADC 模块的驱动中可以阅读 `src/adc/interrupt.rs` 以及 `src/adc/mod.rs`

``` rust
//  src/adc/interrupt.rs
use crate::pac::interrupt;

pub(super) static mut CLOSURE: Option<*const dyn Fn()> = None;

pub fn dispatch() {
    unsafe {
        if let Some(func) = CLOSURE {
            (*func)()
        }
    }
}

// ADC 中断服务函数
#[interrupt]
fn ADC_COMP() {
    // ADC1 的中断 eoc
    critical_section::with(|_cs| {
        dispatch();
    })
}

// src/adc/mod.rs

impl<'d, T: Instance, M: Mode> AnyAdc<'d, T, M> {
    pub fn on_interrupt(
        &mut self,
        events: EnumSet<Event>,
        callback: alloc::boxed::Box<dyn Fn(u16)>,
    ) {
        crate::interrupt::register(
            #[allow(static_mut_refs)]
            unsafe {
                &mut CLOSURE
            },
            alloc::boxed::Box::new(move || {
                callback(T::data_read());
                for e in events {
                    T::event_flag(e);
                }
            }),
        );
        for e in events {
            self.event_config(e, true);
        }
    }
}

```

 ADC 的中断服务函数为：`fn ADC_COMP()`, 中断发生时，进入 `fn dispatch()` 函数，`fn dispatch()` 内部检查存储ADC中断专用的闭包全局变量，当设置了 `Option` 的值后，即注册了闭包，则执行闭包的逻辑。

 `on_interrupt` 函数为 ADC 内部成员函数，负责提供给上层应用接口注入闭包逻辑。 `crate::interrupt::register(...)` 函数实现如下：
 ``` rust
// src/interrupt.rs
use alloc::boxed::Box;

pub fn register(closure: &'static mut Option<*const dyn Fn()>, f: Box<dyn Fn()>) {
    unsafe {
        if let Some(old) = *closure {
            *closure = None;
            let _ = alloc::boxed::Box::from_raw(old as *mut dyn Fn());
        }
        let raw = alloc::boxed::Box::into_raw(f);
        *closure = Some(raw)
    }
}
 ```
 注册函数将会使用 Box 申请堆空间用于存储闭包逻辑，因此可在测试的 main 函数中看到设置堆区的初始化逻辑。要求堆的大小必须保证大于上层接口中的闭包所占用的空间大小。在注册新的闭包时，`register` 内部会管理旧的闭包，即释放旧的堆空间，读者可以尝试注释`let _ = alloc::boxed::Box::from_raw(old as *mut dyn Fn());` 后，多次调用 `on_interrupt` 即可。

 ## 测试结果
 运行:`cargo r --example adc_block_interrupt_closure --no-default-features -F "example"`

注意需要关闭默认的宏`embassy`, 在 `cargo r` 命令后添加 `--no-default-features` 即可
 ``` bash
      Erasing ✔ [00:00:00] [#######################################################################] 16.00 KiB/16.00 KiB @ 193.12 KiB/s (eta 0s )
  Programming ✔ [00:00:01] [#######################################################################] 13.75 KiB/13.75 KiB @ 8.18 KiB/s (eta 0s )    Finished in 1.784s
INFO  8000000
└─ adc_block_interrupt_closure::__cortex_m_rt_main::{closure#0} @ examples/adc_block_interrupt_closure.rs:35
INFO  adc "[0, 1441, 2170, 1394, 2243, 1446, 2172, 1399, 2251, 1442, 2182, 1412, 2261, 0, 0, 0]" sum: 1441 avrage: 1133
└─ adc_block_interrupt_closure::__cortex_m_rt_main::{closure#0} @ examples/adc_block_interrupt_closure.rs:78
INFO  adc "[2210, 1426, 2278, 1370, 2220, 1429, 2258, 1380, 2228, 1435, 2182, 1412, 2261, 1428, 2278, 1362]" sum: 26881 avrage: 1820
└─ adc_block_interrupt_closure::__cortex_m_rt_main::{closure#0} @ examples/adc_block_interrupt_closure.rs:78
INFO  adc "[2268, 1456, 2180, 1402, 2256, 1408, 2202, 1418, 2274, 1367, 2203, 1419, 2278, 1363, 2212, 1430]" sum: 29056 avrage: 1819
└─ adc_block_interrupt_closure::__cortex_m_rt_main::{closure#0} @ examples/adc_block_interrupt_closure.rs:78

 ```

 ## 引用
 [Rust语言圣经(Rust Course)](https://course.rs/advance/functional-programing/closure.html)
 [py32f030-hal](https://github.com/hysonglet/py32f030-hal)