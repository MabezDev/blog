+++
title="Rust on the ESP32 & ESP8266 - Building an ecosystem"
date=2020-09-15
draft=false
[taxonomies]
tags=["rust", "esp32", "hal", "interrupts", "esp8266", "flash"]
+++


Since my [last post, nearly 10 months ago](https://mabez.dev/blog/posts/esp32-rust-svd-pac/) a lot has changed. For one, its not just me working on this any more! Community members are starting to contribute to the various ESP related projects. The extra help meant we have made considerable progress which I will now take you through in this post.

## The compiler

I have been working on cleaning up the `rustc` work, including rebasing regularly to keep up with upstream rustc changes. Recently, I have cleaned up and extracted the patches required to enable `Xtensa` & `esp` development with Rust, firstly to make it easier to rebase onto a newer version of the compiler, but also in the hopes of one day being able to submit these patches upstream when the `Xtensa` target is upstreamed in llvm. You can see the patchset [here](https://github.com/MabezDev/rust-xtensa-patches). On that note, it seems no progress has been made on that front in the last 9 months; whilst Espressif are doing a great job responding to issues and fixing bugs, it seems the patches are stuck in review in llvm. I'm unsure what directions to take to get more attention to this. 

I have also reworked the build process, as well as tagging releases. It is no longer necessary to build LLVM seperately, as I have swapped out the LLVM submodule to the forked one, meaning you can just use normal rust build instructions. Starting now, I will also be tagging releases every time I rebase on upstream master; on top of which I am also offering prebuilt linux (sorry Windows & Mac users) toolchains on the [releases](https://github.com/MabezDev/rust-xtensa/releases/tag/xtensa-v0.2.0) (if you are building for Mac or Windows, please feel free to send me your prebuilt toolchains for upload).

## Tooling improvements

### cargo-espflash

For those of you not familiar with developing on ESP* platforms, there is a flashing tool `esptool.py` which will flash your program to the device. [@icewind1991](https://github.com/icewind1991) began rewriting this tool in Rust, called [espflash](https://github.com/esp-rs/espflash). [@jessebraham](https://github.com/jessebraham) then submitted [a PR which added a cargo interface](https://github.com/esp-rs/espflash/pull/1) to it, meaning we can easily build and flash all in one go, for example flashing the blinky example in the ESP32 HAL:

```bash
$ cargo espflash --chip esp32 --example blinky --target xtensa-esp32-none-elf /dev/ttyUSB0
```

It's really nice to invoke cargo once, make a coffee, and have your code running on the board when you get back! We've been adding more features to the tool, like reading back board info (flash size, CPU revision etc) in [#3](https://github.com/esp-rs/espflash/pull/3), and allowing faster flash speeds for the ESP32 in [#5](https://github.com/esp-rs/espflash/pull/5).

### idf2svd

[@jessebraham](https://github.com/jessebraham) submitted a PR to [idf2svd](https://github.com/MabezDev/idf2svd) to add support for generating ESP8266 SVD's, of which the esp8266-hal is built upon; [if you remeber back to the bonus section of my last post](https://mabez.dev/blog/posts/esp32-rust-svd-pac/), you will be able to use that SVD to debug ESP8266 applications too!

## The runtime crates

After my last post I set to improve the runtime crate to include support for exceptions, which [I started before](https://github.com/esp-rs/xtensa-lx6-rt/pull/6) taking a short break. In that time [@arjanmels](https://github.com/arjanmels) championed it, fully implementing exception and interrupt handling for the lx6 CPU of the ESP32! On top of that, for those of you familiar with the `cortex-m-rt` crates, we implemented the attribute macros for defining `#[interrupt]`/`#[exception]` handlers and the `#[entry]` point to the program. Previously [`xtensa-lx6`](https://github.com/esp-rs/xtensa-lx6) was empty, but we have since implemented (and moved code from rt where neccessary), including the on chip [timers](https://docs.rs/xtensa-lx6/0.2.0/xtensa_lx6/timer/index.html) and a series of [mutex](https://docs.rs/xtensa-lx6/0.2.0/xtensa_lx6/mutex/index.html) implementations for the platform.

Meanwhile [@icewind1991](https://github.com/icewind1991) began writing a runtime crate for the [lx106](https://github.com/icewind1991/xtensa-lx106-rt), the processor in a ESP8266. We found that between the `xtensa-lx` series there were not many differences, confirming what a [core esp-idf developer mentioned](https://github.com/esp-rs/xtensa-lx-rt/issues/5#issuecomment-578419057) previously, therefore we decided to merge the lx6 and lx106 crates producing `xtensa-lx` and `xtensa-lx-rt` with features for each silicon revision.

## The HAL crates

### ESP32

With the runtime crates in good shape, we now had a good basis to build a HAL (Hardware abstraction layer). I started by submitting a simple [GPIO driver for the ESP32](https://github.com/esp-rs/esp32-hal/commit/8a3f2e750335623b551d59419e4e138659cc77aa) implementing the [embedded-hal digital traits](https://github.com/rust-embedded/embedded-hal). Since then the features haven't stopped coming, primarily thanks once again to [@arjanmels](https://github.com/arjanmels). Checkout [the examples](https://github.com/esp-rs/esp32-hal/tree/master/examples) to see what you can do with an ESP32 in pure Rust[^pure-rust]! 

### ESP8266

[@icewind1991](https://github.com/icewind1991) has since started the ESP8266 HAL. One thing that is slowing down development is the fact that alot of the functions in the C SDK are simply binary blobs (or as Espressif refer to them, ROM functions). It means that re-implementing them in Rust requires disassembly  of the binaries and reverse engineering of the assembly to figure out whats going on. Whilst it is undoubtedly fun, it is a lot harder than looking at the source (which we can do with the ESP32). It's coming along nicely though, with support for GPIO, Serial, SPI and much more. Checkout the [esp-rs/esp8266-hal](https://github.com/esp-rs/esp8266-hal) repo. [@jessebraham](https://github.com/jessebraham) has started a BSP (board support package) for the D1 Mini ESP8266 board, built on top of the esp8266-hal, so if you have one of those board lying around check out the [repo](https://github.com/jessebraham/d1-mini)!

## The quickstart

With the compiler changes and HAL's being created, the old quickstart needed some love. First on the list removing some unneeded compiler restrictions, mainly around allowing debug info generation, as the LLVM fork now supports that (See [#23](https://github.com/MabezDev/xtensa-rust-quickstart/pull/23)). The biggest change has been adding runnable examples for both the ESP32 & ESP8266; its now easier than ever to Rustup your Esp microcontroller! I have also added [a brief overview](https://github.com/MabezDev/xtensa-rust-quickstart/blob/04f9fb79e9a6a519a3311bc1e5a5e22af7c93f29/src/main.rs#L1-L55) of how to target a different Xtensa target other than the ESP32 or ESP8266, though I'm not aware of any other boards using this architecture, would be very interested to hear about it if you are aware of any.

## What's next?

* I'd love to see more contributors, as it's easier than ever to contribute, both HAL's are in a good spot to pick a hardware feature and implement it in Rust! 
* Myself and [@arjanmels](https://github.com/arjanmels) are looking [at WiFi/Bluetooth support](https://github.com/esp-rs/esp32-wifi), but haven't had much luck so far. 
* At some point I'd like to start looking into integrating with an existing Rust RTOS, perhaps [tockos](https://www.tockos.org/) or [rtic](https://rtic.rs/0.5/book/en/).

At this point I'd like to say a big thank you to all the contributors who have helped along the way so far!

Finally, if you appreciate the effort going into this, consider joining @davidkern, @DaMouse404 and others [in sponsoring me](http://github.com/mabezdev), it's very much appreciated <3.

<iframe src="https://github.com/sponsors/MabezDev/card" title="Sponsor MabezDev" height="225" width="600" style="border: 0;"></iframe>

<br>

---

<br>

[^pure-rust]: Not quite! We're still using the C bootloader to initialize the flash for now. Note that you can run applications entirely in RAM if you wish.