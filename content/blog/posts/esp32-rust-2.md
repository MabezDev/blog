+++
title="Rust on the ESP32"
date=2020-01-13
draft=false
[taxonomies]
tags=["rust", "esp32", "hal", "svd"]
+++

Since my [last post](https://mabez.dev/blog/posts/esp32-rust/) I've been quietly working on building the humble beginnings of an ecosystem around the esp32 for Rust.

## Summary

The short version of this post is as follows:

* [This PR](https://github.com/MabezDev/xtensa-rust-quickstart/pull/4) means the quickstart repo can now flash and run code without a debugger! Simply using the `flash` or `flash_release` scripts will flash your code and begin running your code
* [idf2svd](https://github.com/MabezDev/idf2svd) has been created which uses the documentation from `esp-idf` to generate svd files fit for consumption by [svd2rust](https://github.com/rust-embedded/svd2rust)
* Based on the generated SVD file, there is now a [PAC (Peripheral access crate)](https://docs.rs/esp32/0.1.0/esp32/) for the ESP32. This means we no longer have to worry about writing values into seemingly random address; instead we get to use the awesome svd2rust API! For example, disabling the rtc watchdog looks like this: 
    ```rust
    fn disable_rtc_wdt(rtccntl: &mut esp32::RTCCNTL) {
        /* Disables the RTCWDT */
        rtccntl.wdtwprotect.write(|w| unsafe { w.bits(WDT_WKEY_VALUE) });
        rtccntl.wdtconfig0.modify(|_, w| unsafe {
            w
            .wdt_stg0()
            .bits(0x0)
            .wdt_stg1()
            .bits(0x0)
            .wdt_stg2()
            .bits(0x0)
            .wdt_stg3()
            .bits(0x0)
            .wdt_flashboot_mod_en()
            .clear_bit()
            .wdt_en()
            .clear_bit()
        });
        rtccntl.wdtwprotect.write(|w| unsafe { w.bits(0x0) });
    }
    ```


## Fixing `xtensa-rust-quickstart`

As noted in my previous post, running rust applications on the esp32 required the use of a JTAG debugger as the board would reset repeatedly. My suspicion was a watch dog reset, but my [initial experiments on disabling the watchdog](https://github.com/MabezDev/xtensa-rust-quickstart/pull/4/commits/d8d6971285d20aacb6db32a68138c58a77fa9efa) did not appear to be working. I decided to inspect the openocd source to see what the debugger does when it connects, it turns out that the esp32 has another 2 timer based watch dogs as seen [being disabled by openocd here](https://github.com/espressif/openocd-esp32/blob/97ba3a6bb9eaa898d91df923bbedddfeaaaf28c9/src/target/esp32.c#L431); after disabling those two watchdogs, its now possible to flash and run code with just the USB cable connected!

## Creating esp32

The [SVD format](http://www.keil.com/pack/doc/CMSIS/SVD/html/index.html) was created for ARM processors, but as it turns out it works very well for other processor architectures too. Espressif (unsuprisingly) don't provide an SVD file, so I would have to create my own; but before I move on to that, what do we actually get from creating and SVD file for the ESP32? Well, it allows us to use [svd2rust](https://github.com/rust-embedded/svd2rust) to generate really nice register access API's. In the last post, all peripheral manipulation was achieved through seemingly random volatile writes at given addresses; of course this still needs to happen, but do we need to care about the address and bitshift values? svd2rust hides all of that behind a [simple, well documented API](https://docs.rs/svd2rust/0.17.0/svd2rust/) which makes developing higher level abstractions far simpler.

To generate the initial SVD, the data has to come from somewhere. Intially @jeandudey put together an SVD including just the GPIO peripheral, but [this approach would be a lot of work](https://github.com/esp-rs/esp32-hal/pull/2#issuecomment-565825098) to generate info for the entire chip. I decided to explore parsing the info from the esp-idf (espresif's C based HAL/libs)code documentation, hence [idf2svd](https://github.com/MabezDev/idf2svd) was born; it does a good job of scraping 95%~ of the data we need but [there are still a few things](https://github.com/MabezDev/idf2svd/issues) that could be improved.

Fortunately a lot of the issues with the generated data can be fixed with the [svd patching tool](https://github.com/stm32-rs/svdtools), there have already been a few PR's to fix missing peripherals that idf2svd [missed](https://github.com/esp-rs/esp32/pull/8) or [clean up register]((https://github.com/esp-rs/esp32/pull/7)) and bitfield names. Any PR's to clean up the SVD are very welcome!

## Bonus: Using cortex-m debug with the ESP32

// TODO


## What's next

// still waiting on espresiff, but esp32-hal

// consider sponsor

