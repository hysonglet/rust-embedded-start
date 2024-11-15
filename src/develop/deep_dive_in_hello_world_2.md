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

基于模板创建新工程。使用命令：`cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart`, 然后输入合适的工程名字即可。

``` text
➜  tmp cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart --name py32f030_hello_world_start
🔧   Destination: /Users/hunter/Desktop/tmp/tmp/py32f030_hello_world_start ...
🔧   project-name: py32f030_hello_world_start ...
🔧   Generating template ...
🔧   Moving generated files into: `/Users/hunter/Desktop/tmp/tmp/py32f030_hello_world_start`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /Users/hunter/Desktop/tmp/tmp/py32f030_hello_world_start
```

## 修改

创建的 工程可能与你的芯片并不完全匹配，因此你可能需要检查以下几个部分：

1. 初步编译，可以正常通过，
```
➜  py32f030_hello_world_start git:(master) ✗ cargo b
    Updating crates.io index
     Locking 31 packages to latest compatible versions
      Adding aligned v0.3.5 (latest: v0.4.2)
      Adding as-slice v0.1.5 (latest: v0.2.1)
      Adding bare-metal v0.2.5 (latest: v1.0.0)
      Adding bitfield v0.13.2 (latest: v0.17.0)
      Adding cortex-m v0.6.7 (latest: v0.7.7)
      Adding cortex-m-rt v0.6.15 (latest: v0.7.5)
      Adding cortex-m-rt-macros v0.6.15 (latest: v0.7.5)
      Adding cortex-m-semihosting v0.3.7 (latest: v0.5.0)
      Adding embedded-hal v0.2.7 (latest: v1.0.0)
      Adding generic-array v0.12.4 (latest: v1.1.0)
      Adding generic-array v0.13.3 (latest: v1.1.0)
      Adding generic-array v0.14.7 (latest: v1.1.0)
      Adding nb v0.1.3 (latest: v1.1.0)
      Adding panic-halt v0.2.0 (latest: v1.0.0)
      Adding r0 v0.2.2 (latest: v1.0.0)
      Adding rustc_version v0.2.3 (latest: v0.4.1)
      Adding semver v0.9.0 (latest: v1.0.23)
      Adding semver-parser v0.7.0 (latest: v0.10.2)
      Adding syn v1.0.109 (latest: v2.0.87)
   Compiling semver-parser v0.7.0
   Compiling typenum v1.17.0
   Compiling cortex-m v0.7.7
   Compiling proc-macro2 v1.0.89
   Compiling version_check v0.9.5
   Compiling unicode-ident v1.0.13
   Compiling nb v1.1.0
   Compiling void v1.0.2
   Compiling vcell v0.1.3
   Compiling syn v1.0.109
   Compiling stable_deref_trait v1.2.0
   Compiling cortex-m-rt v0.6.15
   Compiling bitfield v0.13.2
   Compiling cortex-m-semihosting v0.3.7
   Compiling volatile-register v0.2.2
   Compiling cortex-m v0.6.7
   Compiling nb v0.1.3
   Compiling r0 v0.2.2
   Compiling embedded-hal v0.2.7
   Compiling py32f030_hello_world_start v0.1.0 (/Users/hunter/Desktop/tmp/tmp/py32f030_hello_world_start)
   Compiling panic-halt v0.2.0
   Compiling semver v0.9.0
   Compiling rustc_version v0.2.3
   Compiling generic-array v0.14.7
   Compiling bare-metal v0.2.5
   Compiling quote v1.0.37
   Compiling generic-array v0.13.3
   Compiling generic-array v0.12.4
   Compiling as-slice v0.1.5
   Compiling aligned v0.3.5
   Compiling cortex-m-rt-macros v0.6.15
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 8.98s
```
2. 修改 `memory.x` , 指定 flash 和 ram 信息, 修改如下：
``` bash
diff --git a/memory.x b/memory.x
index b271f22..0f0d381 100644
--- a/memory.x
+++ b/memory.x
@@ -2,9 +2,9 @@ MEMORY
 {
   /* NOTE 1 K = 1 KiBi = 1024 bytes */
   /* TODO Adjust these memory regions to match your device memory layout */
-  /* These values correspond to the LM3S6965, one of the few devices QEMU can emulate */
-  FLASH : ORIGIN = 0x00000000, LENGTH = 256K
-  RAM : ORIGIN = 0x20000000, LENGTH = 64K
+  /* PY32F030K28T6: */
+  FLASH : ORIGIN = 0x08000000, LENGTH = 64K
+  RAM : ORIGIN = 0x20000000, LENGTH = 8K
 }

 /* This is where the call stack will be allocated. */
```

3.  修改编译目标和运行命令
``` bash
diff --git a/.cargo/config.toml b/.cargo/config.toml
index 9709a75..cb7fde1 100644
--- a/.cargo/config.toml
+++ b/.cargo/config.toml
@@ -2,6 +2,9 @@
 # uncomment this to make `cargo run` execute programs on QEMU
 # runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

+[target.thumbv6m-none-eabi]
+runner = 'probe-rs run --chip PY32F030x8'
+
 [target.'cfg(all(target_arch = "arm", target_os = "none"))']
 # uncomment ONE of these three option to make `cargo run` start a GDB session
 # which option to pick depends on your system
@@ -28,8 +31,8 @@ rustflags = [

 [build]
 # Pick ONE of these default compilation targets
-# target = "thumbv6m-none-eabi"        # Cortex-M0 and Cortex-M0+
-target = "thumbv7m-none-eabi"        # Cortex-M3
+target = "thumbv6m-none-eabi"        # Cortex-M0 and Cortex-M0+
+# target = "thumbv7m-none-eabi"        # Cortex-M3
 # target = "thumbv7em-none-eabi"       # Cortex-M4 and Cortex-M7 (no FPU)
 # target = "thumbv7em-none-eabihf"     # Cortex-M4F and Cortex-M7F (with FPU)
 # target = "thumbv8m.base-none-eabi"   # Cortex-M23
```

如上修改后，执行 `cargo r`:
``` bash
➜  py32f030_hello_world_start git:(main) ✗ cargo r
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.01s
     Running `probe-rs run --chip PY32F030x8 target/thumbv6m-none-eabi/debug/py32f030_hello_world_start`
 WARN probe_rs::util::rtt: No RTT header info was present in the ELF file. Does your firmware run RTT?
      Erasing ✔ [00:00:00] [####################################] 4.00 KiB/4.00 KiB @ 53.93 KiB/s (eta 0s )
  Programming ✔ [00:00:00] [#####################################] 2.00 KiB/2.00 KiB @ 3.87 KiB/s (eta 0s )    Finished in 0.63s
```

cargo 将编译工程，然后执行  **`runner`** 配置命令，即使用 `probe-rs` 命令下载固件到芯片，并执行。可以看到，下载完成成后并没有其他日志打印，因此我们需要继续添加打印日志。

4. 添加日志 crate `rtt-target` ，让日志打印像本地端一样简单。修改如下：
``` bash
diff --git a/Cargo.toml b/Cargo.toml
index 1d1df47..a2b897d 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -6,11 +6,13 @@ name = "py32f030_hello_world_start"
 version = "0.1.0"

 [dependencies]
-cortex-m = "0.6.0"
+cortex-m = { version = "0.7.6", features = ["critical-section-single-core"]}
 cortex-m-rt = "0.6.10"
 cortex-m-semihosting = "0.3.3"
 panic-halt = "0.2.0"

+rtt-target = "0.5.0"
+
 # Uncomment for the panic example.
 # panic-itm = "0.4.1"
```
添加 crate 后，在 main.rs 需要添加以下代码：
``` bash
diff --git a/src/main.rs b/src/main.rs
index 7922596..dbeaf9c 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -3,18 +3,26 @@

 // pick a panicking behavior
 use panic_halt as _; // you can put a breakpoint on `rust_begin_unwind` to catch panics
-// use panic_abort as _; // requires nightly
-// use panic_itm as _; // logs messages over ITM; requires ITM support
-// use panic_semihosting as _; // logs messages to the host stderr; requires a debugger
+                     // use panic_abort as _; // requires nightly
+                     // use panic_itm as _; // logs messages over ITM; requires ITM support
+                     // use panic_semihosting as _; // logs messages to the host stderr; requires a debugger

 use cortex_m::asm;
 use cortex_m_rt::entry;

+use rtt_target::{rprintln, rtt_init_print};
+
 #[entry]
 fn main() -> ! {
-    asm::nop(); // To not have main optimize to abort in release mode, remove when you add code
+    // init rtt
+    rtt_init_print!();
+
+    asm::nop();
+
+    rprintln!("Hello, world!");

     loop {
-        // your code goes here
+        // Wait For Interrupt
+        cortex_m::asm::wfi();
     }
 }
```
6. 最后运行，下载完成后马上打印 `hello, world`。在此，你已经成功踏入了嵌入式 Rust 的小门。
``` bash
➜  py32f030_hello_world_start git:(main) ✗ cargo r
   Compiling py32f030_hello_world_start v0.1.0 (/Users/hunter/Desktop/tmp/tttttt/py32f030_hello_world_start)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.10s
     Running `probe-rs run --chip PY32F030x8 target/thumbv6m-none-eabi/debug/py32f030_hello_world_start`
      Erasing ✔ [00:00:00] [##################################] 16.00 KiB/16.00 KiB @ 98.89 KiB/s (eta 0s )
  Programming ✔ [00:00:02] [###################################] 12.25 KiB/12.25 KiB @ 4.21 KiB/s (eta 0s )    Finished in 3.112s
Hello, world!
```

## 查看编译信息
### 固件信息

1. Debug 模式
``` bash
➜  py32f030_hello_world_start git:(main) ✗ cargo size
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.01s
   text	   data	    bss	    dec	    hex	filename
  12464	      0	   1088	  13552	   34f0	py32f030_hello_world_start

➜  py32f030_hello_world_start git:(main) ✗ arm-none-eabi-readelf target/thumbv6m-none-eabi/debug/py32f030_hello_world_start -S
There are 22 section headers, starting at offset 0xbf9e4:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .vector_table     PROGBITS        08000000 0000f4 0000c0 00   A  0   0  4
  [ 2] .text             PROGBITS        080000c0 0001b4 002404 00  AX  0   0  4
  [ 3] .rodata           PROGBITS        080024c4 0025b8 000bec 00  AM  0   0  4
  [ 4] .data             PROGBITS        20000000 0031a4 000000 00   A  0   0  4
  [ 5] .bss              NOBITS          20000000 0031a4 000440 00  WA  0   0  4
  [ 6] .uninit           NOBITS          20000440 0031a4 000000 00  WA  0   0  4
  [ 7] .debug_abbrev     PROGBITS        00000000 0031a4 0024cf 00      0   0  1
  [ 8] .debug_info       PROGBITS        00000000 005673 02d143 00      0   0  1
  [ 9] .debug_aranges    PROGBITS        00000000 0327b6 001ab0 00      0   0  1
  [10] .debug_ranges     PROGBITS        00000000 034266 017248 00      0   0  1
  [11] .debug_str        PROGBITS        00000000 04b4ae 0443c6 01  MS  0   0  1
  [12] .comment          PROGBITS        00000000 08f874 00007b 01  MS  0   0  1
  [13] .ARM.attributes   ARM_ATTRIBUTES  00000000 08f8ef 000032 00      0   0  1
  [14] .debug_frame      PROGBITS        00000000 08f924 0057a8 00      0   0  4
  [15] .debug_line       PROGBITS        00000000 0950cc 0262da 00      0   0  1
  [16] .debug_loc        PROGBITS        00000000 0bb3a6 00027b 00      0   0  1
  [17] .debug_pubnames   PROGBITS        00000000 0bb621 0001e9 00      0   0  1
  [18] .debug_pubtypes   PROGBITS        00000000 0bb80a 000047 00      0   0  1
  [19] .symtab           SYMTAB          00000000 0bb854 0016b0 10     21 271  4
  [20] .shstrtab         STRTAB          00000000 0bcf04 0000e9 00      0   0  1
  [21] .strtab           STRTAB          00000000 0bcfed 0029f4 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  y (purecode), p (processor specific)
```
2. Release 模式
```
➜  py32f030_hello_world_start git:(main) ✗ cargo size --release
    Finished `release` profile [optimized + debuginfo] target(s) in 0.01s
   text	   data	    bss	    dec	    hex	filename
   1992	      0	   1088	   3080	    c08	py32f030_hello_world_start

➜  py32f030_hello_world_start git:(main) ✗ arm-none-eabi-readelf target/thumbv6m-none-eabi/release/py32f030_hello_world_start -S
There are 22 section headers, starting at offset 0x1184c:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .vector_table     PROGBITS        08000000 0000f4 0000c0 00   A  0   0  4
  [ 2] .text             PROGBITS        080000c0 0001b4 0006ec 00  AX  0   0  4
  [ 3] .rodata           PROGBITS        080007ac 0008a0 00001c 00   A  0   0  4
  [ 4] .data             PROGBITS        20000000 0008bc 000000 00   A  0   0  4
  [ 5] .bss              NOBITS          20000000 0008bc 000440 00  WA  0   0  4
  [ 6] .uninit           NOBITS          20000440 0008bc 000000 00  WA  0   0  4
  [ 7] .debug_loc        PROGBITS        00000000 0008bc 0013d4 00      0   0  1
  [ 8] .debug_abbrev     PROGBITS        00000000 001c90 0009bc 00      0   0  1
  [ 9] .debug_info       PROGBITS        00000000 00264c 005097 00      0   0  1
  [10] .debug_aranges    PROGBITS        00000000 0076e3 000340 00      0   0  1
  [11] .debug_ranges     PROGBITS        00000000 007a23 000dc0 00      0   0  1
  [12] .debug_str        PROGBITS        00000000 0087e3 005524 01  MS  0   0  1
  [13] .comment          PROGBITS        00000000 00dd07 000048 01  MS  0   0  1
  [14] .ARM.attributes   ARM_ATTRIBUTES  00000000 00dd4f 000032 00      0   0  1
  [15] .debug_frame      PROGBITS        00000000 00dd84 000680 00      0   0  4
  [16] .debug_line       PROGBITS        00000000 00e404 0024c5 00      0   0  1
  [17] .debug_pubnames   PROGBITS        00000000 0108c9 0001e9 00      0   0  1
  [18] .debug_pubtypes   PROGBITS        00000000 010ab2 000047 00      0   0  1
  [19] .symtab           SYMTAB          00000000 010afc 0005c0 10     21  57  4
  [20] .shstrtab         STRTAB          00000000 0110bc 0000e9 00      0   0  1
  [21] .strtab           STRTAB          00000000 0111a5 0006a5 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  y (purecode), p (processor specific)
```


## 参考

[hysonglet/py32f030_hello_world_start ](https://github.com/hysonglet/py32f030_hello_world_start)

[cortex_m_quickstart - Rust (rust-embedded.org)](https://docs.rust-embedded.org/cortex-m-quickstart/cortex_m_quickstart/)

[probe-rs - probe-rs](https://probe.rs/docs/tools/probe-rs/)

