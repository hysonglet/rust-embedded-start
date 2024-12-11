
# ADC

ADC 的 rust crate也提供了异步和阻塞两种接口方式，简化 ADC的配置和使用。

## 示例：`examples/adc_block.rs`
``` rust
#![no_std]
#![no_main]

use hal::adc::{temperature, vrefence_internal, AdcChannel, AnyAdc, ChannelConfig, Config};
use hal::delay;
use py32f030_hal::{self as hal, mode::Blocking};

// use panic_halt as _;
use {defmt_rtt as _, panic_probe as _};

#[cortex_m_rt::entry]
fn main() -> ! {
    let p = hal::init(Default::default());

    let adc: AnyAdc<_, Blocking> = AnyAdc::new(
        p.ADC,
        Config::default(),
        ChannelConfig::new_multiple_channel_perferred(),
        &[AdcChannel::Channel11, AdcChannel::Channel12],
    )
    .unwrap();

    loop {
        // adc.start();
        let temp = adc.read_block(1000000).unwrap();
        // adc.start();
        let vol = adc.read_block(1000000).unwrap();
        defmt::info!(
            "temp: {}: {}, vol: {}: {}",
            temp,
            temperature(temp),
            vol,
            vrefence_internal(vol)
        );
        delay::delay_s(1);
    }
}

```

## 示例二：`examples/embassy_adc.rs`
``` rust
#![no_std]
#![no_main]

use py32f030_hal::adc::{TemperatureChannel, VRrefChannel};
use py32f030_hal::mode::Async;
use py32f030_hal::{self as hal};

use embassy_executor::Spawner;
use embassy_time::Timer;
use hal::adc::{AnyAdc, ChannelConfig, Config};

// use panic_halt as _;
use {defmt_rtt as _, panic_probe as _};

#[embassy_executor::task]
async fn run() {
    loop {
        Timer::after_secs(2).await;
    }
}

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = hal::init(Default::default());

    let adc: AnyAdc<_, Async> = AnyAdc::new(
        p.ADC,
        Config::default(),
        ChannelConfig::new_exclusive_single(),
        &[],
    )
    .unwrap();

    _spawner.spawn(run()).unwrap();

    loop {
        defmt::info!("temp {}", adc.read(TemperatureChannel).await,);
        defmt::info!("vref {}", adc.read(VRrefChannel).await);
        Timer::after_secs(2).await;
    }
}
```

运行：`cargo r --example embassy_adc --features embassy`