
# 外部中断
在上节中, 我们在按键状态的示例中，在循环中定时读取引脚状态，为了尽可能及时读取引脚的状态，需要不断去查询，因此需要占用大量的 CPU 时间。那么如何提高效率呢？

举个例子，你在家里需要一直去通过猫眼检查是否有人在你家门口，一旦有客人来了，你能快速快门接客，但是大部分时间你去查看猫眼时候并没人，因此就会耽误你做其他事情。
那么，如果你在门上安装一个门铃，当客人来了后按下门铃，你再去看门，这样就无需反复去看猫眼，你就能从查询的任务中解放出来，去做其他重要的事，更重要的是，其他的事也不会阻碍你去响应门铃的行为，也就是说，实时性也变高了。

外部中断就是一个这样的逻辑，当引脚的电平变化或电平边沿动作时，CPU 将会收到外部中断，然后进入到中断服务函数。

这一过程与 C 嵌入式的流程完全一致！一旦开启中断使能，只要相应的通道中断没有屏蔽，在匹配到变化的信号后，CPU 自动跳转到 Rust 的中断服务函数。

## 示例
``` rust
#![no_std]
#![no_main]

use embedded_hal::digital::v2::InputPin;
use hal::exti::ExtiInput;
use hal::gpio::{PinPullUpDown, PinSpeed};
use hal::mode::Async;
use py32f030_hal as hal;
use {defmt_rtt as _, panic_probe as _};

use embassy_executor::Spawner;
use embassy_time::Timer;

#[embassy_executor::task]
async fn run(key: ExtiInput<'static, Async>) {
    loop {
        defmt::info!("wating for key push...");
        key.wait_for_low().await;
        defmt::info!("key pushed {}, and wating for key release", key.is_high());
        key.wait_for_high().await;
        defmt::info!("key released");
    }
}

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = hal::init(Default::default());
    let gpioa = p.GPIOF.split();

    defmt::info!("Example: embassy exti!");

    let key: ExtiInput<_> = ExtiInput::new(gpioa.PF4_BOOT0, PinPullUpDown::No, PinSpeed::Low);
    _spawner.spawn(run(key)).unwrap();

    let mut cnt: u32 = 0;
    loop {
        defmt::info!("high {} ", cnt);
        cnt += 1;
        Timer::after_secs(5).await;
    }
}
```
在以上代码中，使用的是异步的方式查询引脚状态，写法与传统的阻塞式编程方式一样，但实际内部则根据中断来唤醒接口。底层驱动如下：
``` rust
use super::types::*;
use crate::clock::peripheral::PeripheralInterrupt;
use crate::exti::hal::sealed::Instance;
use crate::gpio::{AnyPin, GpioPort};
use crate::pac::interrupt;

use embassy_sync::waitqueue::AtomicWaker;

use core::{future::Future, marker::PhantomData, task::Poll};

const EXIT_GPIO_COUNT: usize = 17;
#[allow(clippy::declare_interior_mutable_const)]
const ATOMIC_WAKE_CONST: AtomicWaker = AtomicWaker::new();
static EXIT_GPIO_WAKERS: [AtomicWaker; EXIT_GPIO_COUNT] = [ATOMIC_WAKE_CONST; EXIT_GPIO_COUNT];

impl Instance for Exti {}

pub(crate) struct Exti;

pub struct ExtiInputFuture<'a> {
    line: Line,
    edge: Edge,
    life: PhantomData<&'a mut AnyPin>,
}

impl<'a> ExtiInputFuture<'a> {
    pub fn new(port: GpioPort, pin: usize, edge: Edge) -> Self {
        let line: Line = pin.into();
        // line 选择
        Exti::exit_channle_select(line, port.into());

        critical_section::with(|_| {
            // 设置上升沿触发条件
            Exti::line_ring_edge(line, edge.is_rising());
            // 设置下降沿的触发条件
            Exti::line_falling_edge(line, edge.is_falling());

            // clear pending bit
            Exti::clear_pending(line);
            Exti::line_pend_enable(line, true);
        });

        Self {
            line,
            edge,
            life: PhantomData,
        }
    }
}

impl<'d> Future for ExtiInputFuture<'d> {
    type Output = ();
    fn poll(
        self: core::pin::Pin<&mut Self>,
        cx: &mut core::task::Context<'_>,
    ) -> core::task::Poll<Self::Output> {
        if !Exti::is_line_pend_enable(self.line) {
            Poll::Ready(())
        } else {
            EXIT_GPIO_WAKERS[self.line as usize].register(cx.waker());
            self.line.enable_interrupt();
            Poll::Pending
        }
    }
}

impl<'d> Drop for ExtiInputFuture<'d> {
    fn drop(&mut self) {
        critical_section::with(|_| {
            if self.edge.is_rising() {
                Exti::line_ring_edge(self.line, false);
            } else if self.edge.is_falling() {
                Exti::line_falling_edge(self.line, false);
            }
            // Exit::line_falling_edge(self.line, false);
            // Exit::line_pend_enable(self.line, false);
        })
    }
}

#[interrupt]
fn EXTI0_1() {
    critical_section::with(|_cs| unsafe { on_gpio_line_irq(0x03) })
}

#[interrupt]
fn EXTI2_3() {
    critical_section::with(|_cs| unsafe { on_gpio_line_irq(0xc0) })
}

#[interrupt]
fn EXTI4_15() {
    critical_section::with(|_cs| unsafe { on_gpio_line_irq(0xfff0) })
}

unsafe fn on_gpio_line_irq(mask: u32) {
    let flag = Exti::block().pr.read().bits() & mask;
    for line in BitIter(flag) {
        Exti::line_pend_enable(Line::from(line as usize), false);
        Exti::clear_pending(Line::from(line as usize));
        EXIT_GPIO_WAKERS[line as usize].wake();
    }
}
```

在以上代码中，有分别有3个外部中断的服务函数：`EXTI0_1`,`EXTI2_3`,`EXTI4_15`。
当相应的通道触发中断后，CPU 执行中断服务函数，设置唤醒标志，退出后调度器会自动唤醒上层任务继续执行。

详细驱动代码请查看：`src/exti/mod.rs`,`src/exti/future.rs`,`src/exti/hal.rs`
