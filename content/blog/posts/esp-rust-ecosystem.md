+++
title="Rust on the ESP32 & ESP8266 - Building an ecosystem"
date=2020-09-15
draft=false
[taxonomies]
tags=["rust", "esp32", "hal", "interrupts", "esp8266", "flash"]
+++

// intro & small recap of work

Since my [last post, nearly 10 months ago](https://mabez.dev/blog/posts/esp32-rust-svd-pac/) a lot has changed. For one, its not just me working on this anymore! Community members are starting to contribute to the various ESP related projects. The extra help meant we have made considerable progress which I will now take you through in this post. The short version of this post is:

* 

## The compiler

// talk about it being easier to build (llvm is now a submodule)
// also offering linux dist builds (sorry windows & mac!)

//TODO is rebase the right term here?

I have been working on cleaning up the `rustc` work, including rebasing semi-regularly to keep up with rustc changes. Recently I have cleaned up and extracted the patches required to enable `Xtensa` & `esp` development with Rust, firstly to make it easier to rebase onto a newer version of the compiler, but also in the hopes of one day being able to submit these patches upstream when the `Xtensa` target is upstreamed in llvm. You can see the patchset [here](https://github.com/MabezDev/rust-xtensa-patches). On that note, it seems no progress has been made on that front in the last 9 months; whilst Espressif are doing a great job responding to issues and fixing bugs, it seems the patches are stuck in review in llvm. I'm unsure what directions to take to get more attention to this.

## The runtime crates

After my last post I set to improve the runtime crate to include support for exceptions, which [I started before](https://github.com/esp-rs/xtensa-lx6-rt/pull/6) taking a short break. In that time [@arjanmels](https://github.com/arjanmels) championed it, fully implementing exception and interrupt handling for the lx6 CPU of the ESP32! On top of that, for those of you familiar with the `cortex-m-rt` crates, we implemented the attribute macros for defining `#[interrupt]`/`#[exception]` handlers and the `#[entry]` point to the program. 

Previously [`xtensa-lx6`](https://github.com/esp-rs/xtensa-lx6) served no purpose, but we have since implemented (and moved code from rt where neccessary), including on chip [timers](https://docs.rs/xtensa-lx6/0.2.0/xtensa_lx6/timer/index.html) and a series of [mutex](https://docs.rs/xtensa-lx6/0.2.0/xtensa_lx6/mutex/index.html) implementations for the platform.

// talk about esp8266: https://github.com/icewind1991/xtensa-lx106-rt, mention issue from iggr about merging the two

// hal crates

## The HAL crates

With the runtime crates in good shape, we now had a good basis to build a HAL (Hardware abstraction layer). I started by submitting a simple [GPIO driver](https://github.com/esp-rs/esp32-hal/commit/8a3f2e750335623b551d59419e4e138659cc77aa) implementing the [embedded-hal digital traits](https://github.com/rust-embedded/embedded-hal). Since then the features haven't stopped coming, primarily thanks once again to [@arjanmels](https://github.com/arjanmels). Checkout [the examples](https://github.com/esp-rs/esp32-hal/tree/master/examples) to see what you can do with an ESP32 in pure Rust[^pure-rust]!

// talk about esp8266 hal too

## The quickstart

With the compiler changes and HAL's being created, the old quickstart needed some love. First on the list

// tooling // esp flash as bonus, idf2svd changes

// look to the future, sponsor msg, contributor shout out

<iframe src="https://github.com/sponsors/MabezDev/card" title="Sponsor MabezDev" height="225" width="600" style="border: 0;"></iframe>

<br>

---

<br>

[^pure-rust]: Not quite! We're still using the C bootloader to initialize the flash for now.