
# 使用闭包优雅处理中断
这一章介绍如何使用闭包优雅处理中断。Rust 使用闭包处理中断的设计模式，目的是让中断处理更加及时、同时让 Rust 中的所有权检查能够顺利通过，也让API的使用更加简洁，符合人体工学设计。

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