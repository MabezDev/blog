+++
title="Rust on the ESP32"
date=2019-08-28
draft=false
[taxonomies]
tags=["rust", "esp32", "llvm"]
+++

In this blog post I'll be going over the steps I took to generate valid binaries for the xtensa architecture and then build some `no_std` code to build
a blinky program for the [ESP32](https://www.espressif.com/en/products/hardware/esp32/overview). You can find the fork [here](https://github.com/MabezDev/rust-xtensa).

## Hacking the compiler

In March 2019, Espressif released their first run at an [llvm fork](https://github.com/espressif/llvm-xtensa) to support the xtensa architecure. Shortly after I got to work bootstrapping rust to use this newly created fork. Now prior to this I'd had no experience with the compiler, fortunately I came across the [RISCV PR](https://github.com/rust-lang/rust/pull/52787) which gave me a rough idea of what was required. Fairly soon after that I was able to generate xtensa assembly from Rust!

The next step was to assemble and link the generated assembly; the llvm fork in it's current state cannot perform object generation, so we must use an external assembler. Luckily Rust allows us to do so by specifying the `linker_flavor` as `gcc` and providing a path to the linker with the `linker` target option, in this case `xtensa-esp32-elf-gcc`. I was now able to create static binaries capable of running on xtensa cpu's, woohoo.

## Limitations

At this point I ran into a few issues. The biggest being that the fork struggled with generating debug info; the external assembler does not support [CFI directives](https://sourceware.org/binutils/docs-2.24/as/CFI-directives.html#CFI-directives) something that all llvm targets must support. Removing CFI directives can easily be removed by preprocessing, but does of course add an extra step. After pushing past that issue, I was still getting relocation linker errors. I opened [an issue](https://github.com/espressif/llvm-xtensa/issues/10) to document my findings in the hopes it can be sorted.


With that said, lets try and get a board to blink an LED. This will be a very basic example, with no exception vectors and will run from IROM (not the external flash).

### Blinky code

```
#![no_std]
#![no_main]

#![feature(asm)]

use xtensa_lx6_rt as _;

use core::panic::PanicInfo;

// GPIO output enable reg
const GPIO_ENABLE_W1TS_REG: u32 = 0x3FF44024;

// gpio output set register
const GPIO_OUT_W1TS_REG: u32 = 0x3FF44008;
// gpio output clear register
const GPIO_OUT_W1TC_REG : u32 = 0x3FF4400C;


const BLINKY_GPIO: u32 = 2; // the GPIO hooked up to the onboard LED
const GPIO_FUNCX_OUT_BASE: u32 = 0x3FF44530;
const GPIO_FUNCX_OUT_SEL_CFG: u32 = GPIO_FUNCX_OUT_BASE + (BLINKY_GPIO * 4);

const IO_MUX_GPIO2_REG: u32 = 0x3FF49040;

#[no_mangle]
fn main() -> ! {

    // set the pin as an output
    unsafe {
        core::ptr::write_volatile(GPIO_ENABLE_W1TS_REG as *mut _, 0x1 << BLINKY_GPIO);
        core::ptr::write_volatile(GPIO_FUNCX_OUT_SEL_CFG as *mut _, 0x100); // 0x100 makes this pin a simple gpio pin - see the technical reference
        
        // TODO select function and drive strength
        // if your led pin doesn't default to GPIO you will need to set this register see technical ref for details
        // core::ptr::write_volatile(IO_MUX_GPIO2_REG as *mut _, BLINKY_GPIO); // GPIO2 function 1 is being a gpio port
    }
    loop {
        unsafe {
            core::ptr::write_volatile(GPIO_OUT_W1TS_REG as *mut _, 0x1 << BLINKY_GPIO);           
        }
        delay2(80_000_000);
        unsafe {
            core::ptr::write_volatile(GPIO_OUT_W1TC_REG as *mut _, 0x1 << BLINKY_GPIO);
        }
        delay2(80_000_000);
    }
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {

    }
}

pub fn delay(clocks: u32) {
    let dummy_var: u32 = 0;
    for _ in 0..clocks {
        unsafe { core::ptr::read_volatile(&dummy_var) };
    }
}

pub fn delay2(clocks: u32) {
    for _ in 0..clocks {
        unsafe { asm!("nop" :::: "volatile") };
    }
}

```
After flashing to the board, we are greeted with a blinking LED! <!-- TODO the full src can be found :xxx -->

<video muted controls style="width: 100%; height:auto">
    <source src="/esp-rust-blinky.webm" type="video/mp4">
</video>

Now I know what most of you are thinking at this point, it's not very Rusty; it contains bundles of unsafe and there are no real abstractions here but it's something to get the ball rolling. Without proper debugging symbols its a bit awkward to develop but hopefully that will change too. You may also note the two delay functions, neither of which have the intented effect I was looking for (no matter the value, the delay seems to stay the same) so thats something to investigate.

## A minimal runtime


## Future work

Once debug info is properly implemented, I hope to start developing an ecosystem of HAL's and drivers similar to the [stm32-rs](https://github.com/stm32-rs) and [nrf-rs](https://github.com/nrf-rs); I've already started the [esp-rs](https://github.com/esp-rs) organization in anticipation. Espressif has started the upstream process, the first ten patches are now in review, there should be an update coming to their fork moving from the older llvm6 to llvm8 (and hopefully some other additions too!).