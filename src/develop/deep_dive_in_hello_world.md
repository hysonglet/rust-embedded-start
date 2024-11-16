
# 深入理解 hello world 例程——基于 hal 库

## 工程解说
`py32f030-hal` crate 为 py32f030 芯片的外设支持库, 提供基本的外设访问接口。

`py32f030-hal` crate 的目录结构如下：
``` bash
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

``` bash
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

``` text
MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 64K
  RAM : ORIGIN = 0x20000000, LENGTH = 8K
}
```

在  `memory.x` 脚本中可以看到，
- Flash 的起始地址：0x08000000， 大小为 64K
- RAM 的起始地址：0x20000000， 大小为 8K
### src

`src` 目录为该库的源文件，通常在单片机外设库中，每个外设作为一个单独的模块。模块的根节点在 lib.rs 中，模块名即为包名，也就是 `py32f030_hal`。
``` bash
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
use {defmt_rtt as _, panic_probe as _};

#[cortex_m_rt::entry]
fn main_fun() -> ! {
    defmt::info!("hello world");
    loop {
        cortex_m::asm::wfe();
    }
}

```

文件的 前行代码是 Rust 语言的属性宏，用于指定编译时的特定行为。感叹号 ！代表该属性在整个  crate 生效。
- `#![no_std]`:  这个属性宏告诉 Rust 编译器在编译目标程序时不使用标准库（`std`）。标准库提供了很多常用的功能，比如文件I/O、错误处理、集合类型等。不使用标准库通常意味着你需要自己提供这些功能，或者使用其他库来替代。
- `#![no_main]`:  这个属性宏告诉 Rust 编译器不自动生成 `main` 函数。在 Rust 中，`main` 函数是程序的入口点。如果你不使用这个属性宏，编译器会自动寻找一个返回 `()` 类型的 `main` 函数。如果你使用了这个属性宏，你需要自己提供一个 `#[entry]` 属性的函数作为程序的入口点。也就是说，应用程序的入口函数名可以不为 `main`！

第4行 `use py32f030_hal as _;` 引用入本 py32f030_hal.  
第5行引入 `defmt_rtt` 和 `panic_probe`。

第6行 `#[cortex_m_rt::entry]` 的属性宏将告诉编译起后面的函数名将作为程序入口。

第8行 `fn main_fun() -> !`, `!`表明函数 main_fun 将永不返回。这样设计避免编译检查警告。

第9 行 `defmt::info!("hello world");` 该语句将会在终端中打印字符串。然后系统进入停止模式。

## 编译
通过命令：`cargo b --example hello_world` 编译程序。首次编译可能耗时较长。
``` bash
➜  py32f030-hal git:(main) ✗ cargo b --example hello_world
warning: unused manifest key: dependencies.embedded-io-async.option
   Compiling proc-macro2 v1.0.85
   Compiling unicode-ident v1.0.12
   Compiling version_check v0.9.4
   Compiling syn v1.0.109
   Compiling semver-parser v0.7.0
   Compiling nb v1.1.0
   Compiling cortex-m v0.7.7
   Compiling void v1.0.2
   Compiling nb v0.1.3
   Compiling thiserror v1.0.61
   Compiling critical-section v1.1.2
   Compiling embedded-hal v0.2.7
   Compiling semver v1.0.23
   Compiling semver v0.9.0
   Compiling defmt v0.3.8
   Compiling rustc_version v0.2.3
   Compiling proc-macro-error-attr v1.0.4
   Compiling proc-macro-error v1.0.4
   Compiling autocfg v1.3.0
   Compiling bare-metal v0.2.5
   Compiling num-traits v0.2.19
   Compiling ident_case v1.0.1
   Compiling byteorder v1.5.0
   Compiling vcell v0.1.3
   Compiling fnv v1.0.7
   Compiling defmt-macros v0.3.9
   Compiling az v1.2.1
   Compiling volatile-register v0.2.2
   Compiling quote v1.0.36
   Compiling syn v2.0.66
   Compiling bitfield v0.13.2
   Compiling cortex-m-rt v0.7.3
   Compiling atomic-polyfill v1.0.3
   Compiling rustc_version v0.4.0
   Compiling bitflags v1.3.2
   Compiling panic-probe v0.3.2
   Compiling defmt-rtt v0.4.1
   Compiling ssd1309 v0.3.0
   Compiling heapless v0.7.17
   Compiling PY32f030xx-pac v0.1.0
   Compiling display-interface v0.4.1
   Compiling embedded-io-async v0.6.1
   Compiling embedded-hal-async v1.0.0
   Compiling embassy-hal-internal v0.1.0
   Compiling gcd v2.3.0
   Compiling fugit v0.3.7
   Compiling embedded-graphics-core v0.3.3
   Compiling float-cmp v0.8.0
   Compiling darling_core v0.20.9
   Compiling hash32 v0.2.1
   Compiling micromath v1.1.1
   Compiling stable_deref_trait v1.2.0
   Compiling embedded-hal v1.0.0
   Compiling embedded-io v0.6.1
   Compiling embedded-graphics v0.7.1
   Compiling fugit-timer v0.1.3
   Compiling display-interface-i2c v0.4.0
   Compiling embassy-futures v0.1.1
   Compiling panic-halt v0.2.0
   Compiling bare-metal v1.0.0
   Compiling cast v0.3.0
   Compiling drop-move v0.1.0
   Compiling cortex-m-rt-macros v0.7.0
   Compiling thiserror-impl v1.0.61
   Compiling darling_macro v0.20.9
   Compiling darling v0.20.9
   Compiling enumset_derive v0.10.0
   Compiling enumset v1.1.5
   Compiling defmt-parser v0.3.4
   Compiling py32f030_hal v0.1.0 (/Users/hunter/mywork/py32/py32f030-hal)
    Finished `dev` profile [optimized + debuginfo] target(s) in 38.96s
```

编译完成后生成的 elf 文件路径为：
- Debug模式: `target/thumbv6m-none-eabi/debug/examples/hello_world`

- Release模式：`target/thumbv6m-none-eabi/debug/examples/hello_world`

常使用size 命令查看 flash 和 ram 占用情况。
``` bash
arm-none-eabi-size target/thumbv6m-none-eabi/debug/examples/hello_world 
   text    data     bss     dec     hex filename
   6344      56    1032    7432    1d08 target/thumbv6m-none-eabi/debug/examples/hello_world
```
在默认情况下，Debug 模式可能会占用较多的空间，但通常在 Release 模式下，则会减少很多。由于目前使用了 `defmt` 作为打印接口，因此会占用较多资源，如果资源限制，可选用其他占用较少的 crate。

## 查看汇编

elf 文件转汇编语言通过命令：
`arm-none-eabi-objdump -d target/thumbv6m-none-eabi/debug/examples/hello_world > debug.asm` 

``` asm
target/thumbv6m-none-eabi/debug/examples/hello_world:     file format elf32-littlearm


Disassembly of section .text:

080000bc <Reset>:
 80000bc:	f000 fe25 	bl	8000d0a <DefaultPreInit>
 80000c0:	4808      	ldr	r0, [pc, #32]	; (80000e4 <Reset+0x28>)
 80000c2:	4909      	ldr	r1, [pc, #36]	; (80000e8 <Reset+0x2c>)
 80000c4:	2200      	movs	r2, #0
 80000c6:	4281      	cmp	r1, r0
 80000c8:	d001      	beq.n	80000ce <Reset+0x12>
 80000ca:	c004      	stmia	r0!, {r2}
 80000cc:	e7fb      	b.n	80000c6 <Reset+0xa>
 80000ce:	4807      	ldr	r0, [pc, #28]	; (80000ec <Reset+0x30>)
 80000d0:	4907      	ldr	r1, [pc, #28]	; (80000f0 <Reset+0x34>)
 80000d2:	4a08      	ldr	r2, [pc, #32]	; (80000f4 <Reset+0x38>)
 80000d4:	4281      	cmp	r1, r0
 80000d6:	d002      	beq.n	80000de <Reset+0x22>
 80000d8:	ca08      	ldmia	r2!, {r3}
 80000da:	c008      	stmia	r0!, {r3}
 80000dc:	e7fa      	b.n	80000d4 <Reset+0x18>
 80000de:	f000 f895 	bl	800020c <main>
 80000e2:	de00      	udf	#0
 80000e4:	20000038 	.word	0x20000038
 80000e8:	20000040 	.word	0x20000040
 80000ec:	20000000 	.word	0x20000000
 80000f0:	20000038 	.word	0x20000038
 80000f4:	080018c8 	.word	0x080018c8

...

0800020c <main>:
 800020c:	b580      	push	{r7, lr}
 800020e:	af00      	add	r7, sp, #0
 8000210:	f000 f800 	bl	8000214 <_ZN11hello_world22__cortex_m_rt_main_fun17h27b2c8ae827133dbE>

08000214 <_ZN11hello_world22__cortex_m_rt_main_fun17h27b2c8ae827133dbE>:
 8000214:	b580      	push	{r7, lr}
 8000216:	af00      	add	r7, sp, #0
 8000218:	f000 febe 	bl	8000f98 <_defmt_acquire>
 800021c:	4803      	ldr	r0, [pc, #12]	; (800022c <_ZN11hello_world22__cortex_m_rt_main_fun17h27b2c8ae827133dbE+0x18>)
 800021e:	f000 fddc 	bl	8000dda <_ZN5defmt6export6header17hdbd91843ccf33f48E>
 8000222:	f000 fee5 	bl	8000ff0 <_defmt_release>
 8000226:	bf20      	wfe
 8000228:	e7fd      	b.n	8000226 <_ZN11hello_world22__cortex_m_rt_main_fun17h27b2c8ae827133dbE+0x12>
 800022a:	46c0      	nop			; (mov r8, r8)
 800022c:	00000003 	.word	0x00000003    

...     

```

细心的读者可以看到汇编的 main 函数里面最终调用了函数 `_ZN11hello_world22__cortex_m_rt_main_fun17h27b2c8ae827133dbE`, 与我们定义的
``` rust
#[cortex_m_rt::entry]
fn main_fun() -> ! 
```

Rust 源码编译后的函数和变量符号变化规律与C++相似，变量在编译后的名字通常按照一定规律进行 manging（名字改编），确保在链接阶段能够正确识别和处理符号，因此模板变量、不同作用域的变量可以允许相同名字。

如果在编写代码时为避免函数名被重命名，可以使用属性`#[no_mangle]`标记避免重命名。通常在以下场景被使用：
- 其他语言绑定，如调用C的函数或变量, 确保接口兼容
- 提供接口给其他语言，如汇编等
- 动态库开发

## 查看符号信息

通常使用 arm-none-eabi-readelf 命令查看编译后的固件的指令格式，大小端，CPU架构，段信息，如中断向量表的偏移、各信息段的大小，方便了解固件各段的具体大小，为优化固件大小提供重要信息。

``` bash
➜  py32f030-hal git:(main) ✗ arm-none-eabi-readelf target/thumbv6m-none-eabi/debug/examples/hello_world -A
Attribute Section: aeabi
File Attributes
  Tag_conformance: "2.09"
  Tag_CPU_arch: v6S-M
  Tag_CPU_arch_profile: Microcontroller
  Tag_ARM_ISA_use: No
  Tag_THUMB_ISA_use: Thumb-1
  Tag_ABI_PCS_R9_use: V6
  Tag_ABI_PCS_GOT_use: direct
  Tag_ABI_FP_denormal: Needed
  Tag_ABI_FP_exceptions: Needed
  Tag_ABI_FP_number_model: IEEE 754
  Tag_ABI_align_needed: 8-byte
  Tag_ABI_align_preserved: 8-byte, except leaf SP
  Tag_CPU_unaligned_access: None
  Tag_ABI_FP_16bit_format: IEEE 754
```

``` bash
➜  py32f030-hal git:(main) ✗ arm-none-eabi-readelf target/thumbv6m-none-eabi/debug/examples/hello_world -S
There are 22 section headers, starting at offset 0x5ef20:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .vector_table     PROGBITS        08000000 010000 0000bc 00   A  0   0  4
  [ 2] .text             PROGBITS        080000bc 0100bc 001d98 00  AX  0   0  4
  [ 3] .rodata           PROGBITS        08001e54 011e54 000738 00  AM  0   0  4
  [ 4] .data             PROGBITS        20000000 020000 000040 00  WA  0   0  8
  [ 5] .gnu.sgstubs      PROGBITS        080025e0 020040 000000 08   A  0   0 32
  [ 6] .bss              NOBITS          20000040 020040 000140 00  WA  0   0  8
  [ 7] .uninit           NOBITS          20000180 020040 000400 00  WA  0   0  4
  [ 8] .defmt            PROGBITS        00000000 020040 000006 00      0   0  1
  [ 9] .debug_loc        PROGBITS        00000000 020046 0036a7 00      0   0  1
  [10] .debug_abbrev     PROGBITS        00000000 0236ed 000f37 00      0   0  1
  [11] .debug_info       PROGBITS        00000000 024624 0121c5 00      0   0  1
  [12] .debug_aranges    PROGBITS        00000000 0367e9 000bd0 00      0   0  1
  [13] .debug_ranges     PROGBITS        00000000 0373b9 002018 00      0   0  1
  [14] .debug_str        PROGBITS        00000000 0393d1 01bd34 01  MS  0   0  1
  [15] .comment          PROGBITS        00000000 055105 000048 01  MS  0   0  1
  [16] .ARM.attributes   ARM_ATTRIBUTES  00000000 05514d 000030 00      0   0  1
  [17] .debug_frame      PROGBITS        00000000 055180 000d1c 00      0   0  4
  [18] .debug_line       PROGBITS        00000000 055e9c 00632f 00      0   0  1
  [19] .symtab           SYMTAB          00000000 05c1cc 001200 10     21 218  4
  [20] .shstrtab         STRTAB          00000000 05d3cc 0000dd 00      0   0  1
  [21] .strtab           STRTAB          00000000 05d4a9 001a76 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  y (purecode), p (processor specific)
```
如上所示，可以看到中断向量表 `vector_table` 的地址为 `0x08000000` ,  `data` 段的起始地址为 `0x20000000` , 与 `memory.x` 定义的一致。

