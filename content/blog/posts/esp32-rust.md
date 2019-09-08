+++
title="Rust on the ESP32"
date=2019-08-28
draft=false
[taxonomies]
tags=["rust", "esp32", "llvm"]
+++

About six months ago, I made a [post on reddit](https://www.reddit.com/r/rust/comments/ar2d3r/espressif_have_finally_released_an_llvm_fork_this/) highlighting the launch of Espressif's llvm xtensa fork. I quickly got to bootstrapping the compiler and soon it was possible to generate valid xtensa binaries with Rust! Shortly after I had to put this on hold as I finished my final year of university, in which I used Rust in my dissertation, an embedded ['smartwatch'](https://github.com/MWatch) (I may write about this in the future if anyone is interested).

Since then I have seen a few posts utilising my fork to run Rust on the [ESP32](https://www.espressif.com/en/products/hardware/esp32/overview) ([see this great write up](https://dentrassi.de/2019/06/16/rust-on-the-esp-and-how-to-get-started/) by ctron, if you haven't already), most of which are building on top of esp-idf which is written in C. Today I'll be discussing the steps I took to generate valid binaries for the xtensa architecture with `rustc` and then write some `no_std` code to build a blinky program for the ESP32 only using Rust!

## Hacking the compiler

In March of 2019, Espressif released their first run at an [llvm fork](https://github.com/espressif/llvm-xtensa) to support the xtensa architecure. Shortly after I got to work bootstrapping Rust to use this newly created fork. Now prior to this I'd had no experience with the compiler, fortunately I came across the [RISCV PR](https://github.com/rust-lang/rust/pull/52787) which gave me a rough idea of what was required. Fairly soon after that I was able to generate xtensa assembly from Rust!

The next step was to assemble and link the generated assembly; the llvm fork in it's current state cannot perform object generation, so we must use an external assembler. Luckily Rust allows us to do so by specifying the `linker_flavor` as `gcc` and providing a path to the linker with the `linker` target option, in this case `xtensa-esp32-elf-gcc`. I was now able to create static binaries capable of running on xtensa cpu's, woohoo.

## Blinky code

Now lets try and get a board to blink an LED. First off, we need out basic program structure. `xtensa_lx6_rt` does most of the heavy lifting in this respect, we simply need to define an entry point and the panic handler.

```rust
#![no_std]
#![no_main]


use xtensa_lx6_rt as _;

use core::panic::PanicInfo;

/// Entry point - called by xtensa_lx6_rt after initialisation
#[no_mangle]
fn main() -> ! {
    loop {}
}

/// Simple panic handler
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

Now lets add some register definitions for the peripherals we want to use. For the blinky program, we will need to control the GPIO peripheral.

```rust
/// GPIO output enable reg
const GPIO_ENABLE_W1TS_REG: u32 = 0x3FF44024;

/// GPIO output set register
const GPIO_OUT_W1TS_REG: u32 = 0x3FF44008;
/// GPIO output clear register
const GPIO_OUT_W1TC_REG : u32 = 0x3FF4400C;

const BLINKY_GPIO: u32 = 2; // the GPIO hooked up to the onboard LED

/// GPIO function mode
const GPIO_FUNCX_OUT_BASE: u32 = 0x3FF44530;
const GPIO_FUNCX_OUT_SEL_CFG: u32 = GPIO_FUNCX_OUT_BASE + (BLINKY_GPIO * 4);
```

Using these definitions it should be possible to change the gpio for your board (provided its connected to pins 0-32) by changing the `BLINKY_GPIO`; for my board (NODEMCU ESP-32S) it was GPIO2.

### Initialisation

Next lets setup the pin as a GPIO output. Firstly, for pins 0-32, its simply a case of setting a bit in the GPIO ouput enable register. Secondly the pin has to be configured in GPIO mode. There are not enough pins for all the possible peripherals in the chip, to combat this each pin can have multiple function modes. In the case of the ESP32, each pin has up to 256 different functions, although not all are mapped. To put the pin in GPIO mode, we simply need to write 256 (0x100) to the function select register. After issuing those two register writes, we should be able to turn on the GPIO by setting the relevant bit inside the GPIO set register[^2].

```rust
#[no_mangle]
fn main() -> ! {

    // configure the pin as an output
    unsafe {
        core::ptr::write_volatile(GPIO_ENABLE_W1TS_REG as *mut _, 0x1 << BLINKY_GPIO);
        // 0x100 makes this pin a simple gpio pin - see the technical reference for more info
        core::ptr::write_volatile(GPIO_FUNCX_OUT_SEL_CFG as *mut _, 0x100); 
    }
    // turn on the LED
    unsafe {
        core::ptr::write_volatile(GPIO_OUT_W1TS_REG as *mut u32, 0x1 << idx);           
    }
    loop {}
}
```

### Delaying

For the next stage of our blinky program, we need a way to delay; a simple approach could use `for` loop like so.

```rust
pub fn delay(clocks: u32) {
    let dummy_var: u32 = 0;
    for _ in 0..clocks {
        unsafe { core::ptr::read_volatile(&dummy_var) };
    }
}
```
We add the volatile read so that the compiler doesn't optimise our delay away. The problem with this approach is that depending of the optimisation level, the number of clock cycles each iteration of the loop changes. We need a cycle accurate way of delaying, fortunately the ESP32 has an internal clock counting register which can be accessed with the read special register `rsr` instruction. Now are delay function looks like this.

```rust
/// cycle accurate delay using the cycle counter register
pub fn delay(clocks: u32) {
    // NOTE: does not account for rollover
    // ommitted: the asm to read the ccount
    let target = get_ccount() + clocks;
    loop {
        if get_ccount() > target {
            break;
        }
    }
}
```

Now we have cycle accurate counting we can delay for one second by waiting for the number of cycles the processor will do in one second. The default clock speed on most ESP boards is 40mhz, hence waiting for 40 million cycles equates to a one second delay.

Bringing the snippets together we now have `main` that looks like this.

```rust
#[no_mangle]
fn main() -> ! {

    // configure the pin as an output
    unsafe {
        core::ptr::write_volatile(GPIO_ENABLE_W1TS_REG as *mut _, 0x1 << BLINKY_GPIO);
        // 0x100 makes this pin a simple gpio pin - see the technical reference
        core::ptr::write_volatile(GPIO_FUNCX_OUT_SEL_CFG as *mut _, 0x100); 
    }

    loop {
        set_led(BLINKY_GPIO, true);
        delay(CORE_HZ);
        set_led(BLINKY_GPIO, false);
        delay(CORE_HZ);
    }
}
```

After flashing to the board, and firing up our JTAG debugger[^1], we are greeted with a blinking LED!

<video muted controls style="width: 100%; height:auto">
    <source src="/esp-rust-blinky.webm" type="video/mp4">
</video>

The full source can be found in the [the xtensa quickstart repo](https://github.com/MabezDev/xtensa-rust-quickstart) if you wish to try it for yourself.

Now I know what most of you are thinking at this point, it's not very Rusty; it contains bundles of unsafe and there are no real abstractions here, and you are right; but it's something to get the ball rolling.

## Limitations

There are a few issues, the biggest being that the fork struggled with generating debug info; the external assembler does not support [CFI directives](https://sourceware.org/binutils/docs-2.24/as/CFI-directives.html#CFI-directives) something that all llvm targets must support. CFI directives can easily be removed with some preprocessing, but does of course add an extra step. After pushing past that issue, I was still getting relocation linker errors. I opened [an issue](https://github.com/espressif/llvm-xtensa/issues/10) to document my findings in the hopes it can be sorted.

## Future work

Once the debuginfo issue is sorted, I hope to start developing an ecosystem of HAL's and drivers similar to the [stm32-rs](https://github.com/stm32-rs) and [nrf-rs](https://github.com/nrf-rs); I've already started the [esp-rs](https://github.com/esp-rs) organization which is where `xtensa-lx6-rt` currently resides. Espressif has started the upstream process, the first ten patches are now in review, there should be an update coming to their fork moving from the older llvm6 to llvm8 (and hopefully some other additions and fixes too!).

## Links

- [xtensa-quickstart](https://github.com/MabezDev/xtensa-rust-quickstart) - A quickstart project for using Rust on xtensa
- [rust-xtensa](https://github.com/MabezDev/rust-xtensa) - The xtensa fork of Rust
- [github](https://github.com/MabezDev) - My github

<br/>

---

<br/>

[^2]: This is not always the case if the pin the LED is connected to does not default to the correct iomux function, see the quickstart example for more info.

[^1]: This step is actually very important, currently the board will reset endlessly on boot (I assume due to the watch dog not being disabled), but launching with the debugger works as expected.