+++
title = "My Week in Code #24"
description = "Updates regarding PocketBeagle 2 examples, BeagleBoard Rust Imager and Rust Std for UEFI"
date = "2025-03-09T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "pocketbeagle2", "rust", "uefi"]
+++

Hello everyone. A typical work week. Let's go over everything.

# BeagleBoard Rust Imager

I have been planning to make [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs) more modular, and extract some of its functionality to other crates. This was to clean up bb-imager (library that powers the GUI and CLI), which now had a lot of platform specific functionality.

The cleanup also helped get some significant performance improvements, which is always nice to see. Let's go over the crates that are now in [crates.io](https://crates.io/).

## bb-drivelist

I have been using a patched version of [rs-drivelist](https://crates.io/crates/rs-drivelist) for a while in bb-imager. This was because the version published in [crates.io](https://crates.io/) did not have macOS support. I did contribute the [patches](https://github.com/ir1keren/rs-drivelist/pull/3) upstream, and they were merged. However, it seems that due to some [personal issues](https://github.com/ir1keren/rs-drivelist/issues/4#issuecomment-2393393168), the original author cannot push a new release.

Additionally, I also wanted to drop dependencies on the old [json](https://crates.io/crates/json) crate, and instead use a serde based solution. So I finally decided to make a hard fork and publish a new [release](https://crates.io/crates/bb-drivelist).

It would be nice if I could also drop [wiapi](https://crates.io/crates/winapi) in favor of [windows](https://crates.io/crates/windows), but that seems a bit more work that I have the time for right now. So if anyone wants to take a crack at it, feel free to try.

## bb-flasher-bcf

Support for [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) was the main reason for the creating BeagleBoard Rust Imager. It has been present for a while, so, was straightforward to extract. The [bb-flasher-bcf crate](https://crates.io/crates/bb-flasher-bcf) contains support for both [CC1352P7](https://www.ti.com/product/CC1352P7) and [MSP430F5503](https://www.ti.com/product/MSP430F5503), which serves as the USB to UART bridge.

I have also made the flashing sync, since I feel that most of the flashing should be done in separate threads. The flashing still includes support for real-time progress and early cancellation.

The library has only been tested on Linux and Windows.

## bb-flasher-sd

For the longest time, SD card flashing in BeagleBoard Rust Imager was slower than Raspberry Pi Imager. However, with all the cleanup while creating [bb-flasher-sd](https://crates.io/crates/bb-flasher-sd), the flashing got a huge speed-up, and should now be similar to Raspberry Pi Imager. Additionally, I have drastically cut down on the stack usage, so that should also help.

Finally, I have moved SD card flashing to a separate OS thread instead of tokio thread. This is because it is better to use OS threads for such long-running tasks.

The library should allow SD card flashing of OS images (even non Beagle ones) from Linux, Windows and macOS.

# PocketBeagle 2 Examples

More work went into improving [PocketBeagle 2 Examples](https://openbeagle.org/beagleboard/vsx-examples). They are almost at the point where they can serve as nice getting started to teach new people about embedded programming.

## Sysfs LED

The original LED abstraction in Beagle Helper library only supported GPIO pins. However, some LEDs (like USR_LED{1-4} in PocketBeagle 2) are defined in the devicetree and use the Kernel LED driver.

I have added support in LED abstraction to allow using LEDs defined in the devicetree using the sysfs entry. I have also modified the PocketBeagle 2 blinky example to use USR_LED4, since it seemed better to not require external LED for a simple blinky example.

## Update README

The README of all the examples for PocketBeagle 2 have been greatly improved. They now follow a common structure with the following sections:

1. **Goal:** Goals of the example
2. **Overview:** Details regarding the background of the example.
3. **Challenges:** Some challenges to get better acquainted with the example.

They should provide a better start for creating tutorials using the examples.

## CI Testing

All the Rust examples were already being built in CI to ensure that at least everything builds fine. However, I was not certain how to do it for an interpreted language like Python. However, I have now started using [pylint](https://pypi.org/project/pylint/), which seems to provide at least a basic level of guarantees regarding all python examples.

# Rust Std for UEFI

I am still waiting on [DevicePathNode PR](https://github.com/rust-lang/rust/pull/137424) to move forward with File I/O implementation. However, I have decided to start pushing some other independent PRs to well, maybe speed things up.

## OwnedEvent Abstraction

Since [Service Binding Protocol PR](https://github.com/rust-lang/rust/pull/137477) was merged last week, the next abstraction required for UEFI networking implementation is Events. This is because all UEFI networking protocols are non-blocking, and thus need the use of Events to be used in a blocking fashion.

Since the rudimentary pieces for events already existed to handle Exit Boot Services event, I just replaced it to use a more strict [OwnedEvent](https://github.com/rust-lang/rust/pull/138236) type, which closes the event on drop. Once it is merged, I will continue the work on implementing TCP4 socket connect.

## Implement basic File I/O Types

Since the other pending PRs make it somewhat hard to work on a lot of other possible parts (I do not want to have to rebase the PRs after each merge), I ended up adding the basic File I/O types such as `FileType`, `FilePermissions` and `FileAttr`. It should help reduce future time waiting around for the basic abstractions to be merged.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs)
- [PocketBeagle 2 Examples](https://openbeagle.org/beagleboard/vsx-examples)
- [bb-drivelist](https://crates.io/crates/bb-drivelist)
- [bb-flasher-bcf](https://crates.io/crates/bb-flasher-bcf)
- [bb-flasher-sd](https://crates.io/crates/bb-flasher-sd)
- [Rust std for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
