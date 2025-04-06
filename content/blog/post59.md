+++
title = "My Week in Code #28"
description = "Updates regarding BeagleBoard Rust Imager and Rust std for UEFI"
date = "2025-04-06T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "uefi", "microblocks", "pocketbeagle2"]
+++

Hello everyone. A typical work week. Let's go over everything.

# BeagleBoard Rust Imager

## BeagleConnect Freedom Fixes

My last updates seem to have caused BeagleConnect Freedom (both CC1352P7 and MSP430) flashing to break. This was reported by @OhioMeasurementDevices on Discord. I have pushed the fixes to crates.io as [version 1.0.2](https://crates.io/crates/bb-flasher-bcf). Additionally, I have also yanked the broken version (1.0.1).

## Release v0.0.5

I have also created a new release for [bb-imager-rs](https://github.com/beagleboard/bb-imager-rs/releases/tag/v0.0.5), which has BeagleConnect Freedom Flashing working again. Additionally, it also fixes links to MicroBlocks firmware, which were broken previously due to some SHA256 problems.

# Zephyr Support for PocketBeagle 2 M4 Core

PocketBeagle 2 features the TI AM62x SoC based around an Arm Cortex-A53 multicore cluster with an Arm Cortex-M4F microcontroller, Imagination Technologies AXE-1-16 graphics processor (from revision A1), and TI programmable real-time unit subsystem microcontroller cluster coprocessors.

The [PR](https://github.com/zephyrproject-rtos/zephyr/pull/87997) brings [ZephyrRTOS](https://zephyrproject.org/) support for the Arm Cortex-M4F microcontroller. This core can be used to offload time-sensitive workflows (PWM bit-banging, Light handling, etc) allowing the A53 cores with Linux to focus on other heavier tasks.

Only UART support is present in the initial PR. However, I will slowly be adding support for other peripherals to be used from the M4F core.

# Migrate MicroBlocks to GitHub

As might be visible, there has been an effort to move a lot of BeagleBoard.org development to [GitHub](https://github.com/beagleboard). This week, I migrated [MicroBlocks](https://github.com/beagleboard/microblocks-zephyr) to our GitHub. Since the CI for MicroBlocks was already broken on [openbeagle.org](https://openbeagle.org/), it just felt like a good project to migrate.

# PocketBeagle 2 Examples

## Migrate to GitHub

The development of PocketBeagle 2 Examples will mostly happen on [GitHub](https://github.com/beagleboard/vsx-examples) now. Hopefully, this will make the examples more discoverable.

## Make the accelerometer example less abstracted

In a recent discussion, it was decided that the examples should try more to teach using Linux APIs instead of just how to use a library to access various peripherals. So as a start, I have added a small abstraction to make interacting with the sysfs entries a bit easier, while keeping things mostly transparent.

The accelerometer example was the first to be switched to the sysfs abstraction. The full diff can be found in the [PR](https://github.com/beagleboard/vsx-examples/pull/3).

## Make the light-sensor example less abstracted

Following the footsteps of the accelerometer example, I have also switched the light-sensor example to use the sysfs abstraction. The full diff can be found in the [PR](https://github.com/beagleboard/vsx-examples/pull/4).

## Make blinky example less abstracted

The blinky example was a bit different since it is the first example that requires writing to a sysfs entry. It is still in the draft stage, waiting for review from everyone, but can be found [here](https://github.com/beagleboard/vsx-examples/pull/5).

# Intial TCP4 socket support for UEFI

I have finally created a [PR](https://github.com/rust-lang/rust/pull/139254) to add initial support for TCP sockets to Rust std for UEFI. It marks the start of UEFI networking in Rust std and is required for running Rust test suite on UEFI using the `remote-test-server` and `remote-test-client` setup as described [here](https://rustc-dev-guide.rust-lang.org/tests/running.html#running-tests-on-a-remote-machine).

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs)
- [PocketBeagle 2 Examples](https://github.com/beagleboard/vsx-examples)
- [Rust std for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
