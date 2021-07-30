+++
title="Rust on Espressif chips"
date=2021-07-28
draft=false
[taxonomies]
tags=["rust", "espressif", "personal"]
+++

It's been a while since my last post. I have some exciting news to share before going over the progress over the last nine months. I will be joining Espressif full time to work on Rust support for all of Espressif's chips, past, present and future! I am very excited to join Espressif and improve the Rust ecosystem surrounding their chips!

### Organisation

The forked compiler with esp support has been moved into the `esp-rs` organisation, along with some idf support crates - more on that shortly! [Discussions](https://github.com/esp-rs/rust/discussions) have been enabled on the `esp-rs/rust` repository, and for the last month Espressif have been hosting weekly community meetings focusing on pushing Rust support forward. If you'd like to discuss something and attend, drop a comment in the latest thread and request an invite!

### The Rust standard library on Espressif chips

[@ivmarkov](https://github.com/ivmarkov) been working very hard on porting the Rust STD library to `esp-idf`, the C based development environment for Espressif chips. `esp-idf` has a newlib environment which can be used to build the Rust library on top of. [@ivmarkov](https://github.com/ivmarkov) has put together [rust-esp32-std-hello](https://github.com/ivmarkov/rust-esp32-std-hello) demo repo to show off the capabilities, which include WiFi, threads and driving a display using the `esp-idf`'s drivers. Extremely cool stuff! There are a few teething issues, which you can track [here](https://github.com/esp-rs/rust/issues/57) but hopefully we will be in a position to start upstreaming these patches soon.

To suppliment the standard library for Espressif chips, [`esp-idf-hal`](https://github.com/esp-rs/esp-idf-hal) has been updated and moved into the `esp-rs` organisation. This crate has API's to use peripherals and hardware that do not have abstractions in the standard library, for example the SPI peripheral. Along with the HAL, [`esp-idf-svc`](https://github.com/esp-rs/esp-idf-svc) has also been added, which is an implementation of [`embedded-svc`](https://github.com/esp-rs/embedded-svc) traits for Espressif chips. The goal behind `embedded-svc` is to provide abstractions around higher level embedded services, WiFi, Ping, HTTPD, NVS etc.

### Bare metal

Whilst the biggest changes have been around esp standard library developments, bare metal support has not been forgotten! A few months I got my hands on a `esp32c3`, a `RISC-V` based chip; because upstream Rust already supports `RISC-V` putting together a rudimentary bare metal example was [quite simple](https://github.com/MabezDev/esp32c3-experiments)!

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Blinky program written in <a href="https://twitter.com/hashtag/rustlang?src=hash&amp;ref_src=twsrc%5Etfw">#rustlang</a>, running on the RISC-V powered <a href="https://twitter.com/hashtag/esp32c3?src=hash&amp;ref_src=twsrc%5Etfw">#esp32c3</a> from <a href="https://twitter.com/hashtag/espressif?src=hash&amp;ref_src=twsrc%5Etfw">#espressif</a> . <a href="https://t.co/9qfqZk1WYq">pic.twitter.com/9qfqZk1WYq</a></p>&mdash; Scott Mabin (@mabezzzzz) <a href="https://twitter.com/mabezzzzz/status/1391500605915607049?ref_src=twsrc%5Etfw">May 9, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Whilst the program above worked, much like [my first ever post](https://mabez.dev/blog/posts/esp32-rust/) it isn't very "Rusty". Fortunately Espressif have recently release [a preliminary SVD file for the esp32c3](https://github.com/espressif/svd/tree/esp32c3), the first of its kind from Espressif which is super exciting! This means we can feed the SVD into `svd2rust` (see my other post about doing this for the esp32 [here](https://mabez.dev/blog/posts/esp32-rust-svd-pac/)) and generate a peripheral access crate (PAC) for the esp32c3 and build abstractions on top of that. With official SVD's coming out I hope that we can integrate that into the current bare metal [`esp32-hal`](https://github.com/esp-rs/esp32-hal), turning it into a mono repository supporting all of Espressif's chips in pure Rust.

It also means I can finally kill off the abomination which is [`idf2svd`](https://github.com/MabezDev/idf2svd) :D.

### What's next?

We should be able to start upstreaming some of the RISC-V based changes soon! With official SVD's coming out, bare metal esp32 support should start moving forward again too.

