+++
title = "My Week in Code #6"
description = "Updates regarding BeagleConnect Freedom, MicroPython, BeagleBoard Rust Imager and Rust std for UEFI"
date = "2024-10-20T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "zephyr", "beagleconnect-freedom", "rust", "uefi"]
+++

Hello everyone. Another light week here. Let's go over everything.

# MicroPython

## Initial BeagleConnect Freedom Support

While we have had [MicroPython](https://micropython.org/) firmware for [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) in the past, the work was out of tree. This made maintenance and collaboration hard to do. There were a few reasons for having the support out of tree:

1. Zephyr support for BeagleConnect Freedom was primarily out of tree, with just basic support in mainline.
2. MicroPython only supported Zephyr v3.1.0

Due to my work, [BeagleConnect Freedom supports almost everything in mainline Zephyr. Additionally, due to the fantastic [work](https://github.com/micropython/micropython/pull/9335) by [Maureen Helm](https://github.com/MaureenHelm) MicroPython now supports Zephyr v3.7.0. So, it seemed like a good time to have support for BeagleConnect Freedom in mainline MicroPython. Here is the link to the merged [PR](https://github.com/micropython/micropython/pull/15959). It does not contain more advanced features like ADC and PWM right now, but those are also on the way.

## PWM support for Zephyr Port

While working on adding PWM support for BeagleConnect Freedom in MicroPython, I discovered that the Zephyr port of MicroPython did not support PWM. So, I ended up working on adding PWM support. Here is the [PR](https://github.com/micropython/micropython/pull/16046).

# MicroBlocks v2.0

MicroBlocks is a blocks programming language for physical computing inspired by Scratch. BeagleConnect Freedom has had good support for MicroBlocks, as outlined in a [prior post](@/blog/post31.md).

Recently, [MicroBlocks v2.0](https://microblocks.fun/blog/2024-10-19-microblocks-2.0-launch/) was released in pilot and thankfully, BeagleConnect Freedom works with the newest firmware perfectly, without any real change in the Zephyr port.

# Arduino Module for Zephyr

Due to the recent cleanup of the defaults in the device tree of BeagleConnect Freedom, ADC and PWM are supported without any overlays. So just a simple cleanup of Arduino module overlay. Here is the [PR](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core/pull/124).

# BeagleBoard Rust Imager

The v0.0.1 release of the BeagleBoard Rust imager took place last week. It has the following features:

1. Platforms:
    1. Linux
        1. GUI Appimage
        2. CLI binary
    2. Windows
        1. GUI Appimage
        2. CLI binary
    3. macOS
        1. CLI binary
2. Boards
    1. Generic Linux (BeaglePlay, Beagle AI64, BeagleY-AI,  BeagleV-Fire, BeagleBone Black, etc)
    2. BeagleConnect Freedom
    3. BeagleConnect Freedom MSP430

It is basically at feature parity with the original [BeagleBoard Imager](https://www.beagleboard.org/bb-imager) based on Raspberry Pi and supports additional boards like BeagleConnect Freedom. Here is the [release page](https://openbeagle.org/ayush1325/bb-imager-rs/-/releases/v0.0.1).

# Rust std for UEFI

The Rust std for UEFI endeavor has picked up steam again, with some of the old PRs getting merged. I have the following support in the pipeline:

1. File I/O
2. TCP
3. UDP

Let's hope that I can get enough free time to upstream everything by the end of this year. However, I will not make any promises since I can only work on this in a limited capacity.

## Environment Variables

Environment variables support for UEFI was merged this week. It uses UEFI Shell protocol behind the scenes since environment variables are not applicable in most other situations. . Here is the [merged PR](https://github.com/rust-lang/rust/pull/127462).

There is also a plan to introduce support for full UEFI variables under `std::os::uefi`, but that's probably far in the future. The current goal is first to reach a point where running Rust std tests is possible.

## File I/O

With most of the pieces in place, I will start working on File I/O support again. Here is the [draft PR](https://github.com/rust-lang/rust/pull/129700), but I have a much more advanced version locally.

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom)
- [My Rust std for UEFI open PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
- [MicroPython](https://micropython.org/)
