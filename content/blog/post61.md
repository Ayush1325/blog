+++
title = "My Week in Code #30"
description = "Updates regarding PocketBeagle 2, BeagleBoard Rust Imager and Rust std for UEFI"
date = "2025-04-20T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "uefi", "pocketbeagle2"]
+++

Hello everyone. A typical work week. Let's go over everything.

# PocketBeagle 2 M4 GPIO Support

Initial [ZephyrRTOS support for PocketBeagle 2 M4 core](https://docs.zephyrproject.org/latest/boards/beagle/pocketbeagle_2/doc/index.html) was merged last week. This only included tested support for UART (RX: `P2.05`, TX: `P2.07`). It is possible to enable GPIO and I2C using overlays, but there are no examples at the moment.

I have created a new [PR](https://github.com/zephyrproject-rtos/zephyr/pull/88731) that enables GPIO in the blinky example (pin `P2.09`). It should serve as a good example of how to use GPIO from the M4 core.

It is important to note that to use any pins from the M4 core, they must first be disabled in the Linux kernel devicetrees. Hopefully, once an upstream connector driver exists, this can be fixed using a special cape for M4 GPIOs, but for now, the user needs to investigate it.

I am also planning to add ZephyrRTOS support for the A53 cores as soon as M4 core support reaches a stable point. Additionally, I also plan to add all possible pinmuxes to the Zephyr devicetree, similar to how we have it in the Linux devicetree. This should allow easier usage of these pins using overlays. Maybe we can even have a GUI to generate overlays at some point.

# PocketBeagle 2 Connector Driver

Since there seem to be some real usage concerns regarding [export-symbols](https://lore.kernel.org/devicetree-spec/20250415122453.68e4c50f@bootlin.com/T/#m591e737b48ebe96aafa39d87652e07eef99dff90), I have been working on an initial PocketBeagle 2 connector driver. To have some fun, I decided to write the driver in Rust, which in the hindside ended up being a bit more work than initially expected.

The current driver is pretty simple. There are some simple principles I am following:

1. Anything attached to the connector is a single cape. It does make non-cape usage a bit more involved right now, but it should be fixable with a simple GUI to generate the required overlay using a few simple toggles.
2. The cape devicetree changes should be concentrated on the connector node. This should help with isolation, and curb the concerns regarding security.
3. Load overlays from `/lib/firmware`. Should help with security concerns upstream.

There is no support for reading cape EEPROM right now. Instead, some sysfs entries are provided to allow adding and removing a cape using the devicetree overlay (present in `/lib/firmware`) name.

Since it is written in Rust, I ended up having to write a lot of Rust abstractions (devicetree overlay application, [sysfs](https://docs.kernel.org/filesystems/sysfs.html), etc). I will start with sending patches for the abstractions first since I am a bit unsure if all the assumptions made in my abstractions are correct. Check out my [working tree](https://github.com/Ayush1325/linux/tree/b4/beagle-cape) for the latest status.

# BeagleBoard Rust Imager

Some bugs were reported in the BeagleBoard Rust Imager which have been fixed now:

- [Allow FAT16 BOOT Partition](https://github.com/beagleboard/bb-imager-rs/pull/28): This is used in some [BeagleBone Black](https://www.beagleboard.org/boards/beaglebone-black) images.

Additionally, thanks to the [work by Sahil Jaiswal](https://github.com/beagleboard/bb-imager-rs/pull/25), the UI now uses an indeterminate progress bar for non-progress states.

# Overhaul Rust UEFI Time

Originally, the implementation of SystemTime for UEFI was using `Duration`, which was anchored to 1970-01-01-00:00:00 ([UNIX EPOCH](https://en.wikipedia.org/wiki/Unix_time)). This meant that date-time before this was simply unrepresentable in the current implementation.

I have rewritten the implementation with 1900-01-01-00:00:00 as the anchor. This date was chosen, since it is the earliest date supported by UEFI internally, according to the [UEFI specification](https://uefi.org/specs/UEFI/2.11/). Additionally, some additional checks were added to ensure that the results of calculations on SystemTime is representable by UEFI time.

Checkout the [PR](https://github.com/rust-lang/rust/pull/139806) for more details.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs)
- [Rust std for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
- [Beagle Cape driver Tree](https://github.com/Ayush1325/linux/tree/b4/beagle-cape)
