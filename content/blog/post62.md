+++
title = "My Week in Code #31"
description = "Updates regarding PocketBeagle 2, BeagleBoard Rust Imager and Rust std for UEFI"
date = "2025-04-27T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "uefi", "pocketbeagle2"]
+++

Hello everyone. A typical work week. Let's go over everything.

# Zephyr support for PocketBeagle 2

PocketBeagle 2 is a single board computer with AM62xx soc, and the primary supported OS is Linux. We provide Debian-based images for the board, with initial support for [Armbian](https://www.armbian.com/pocketbeagle-2/). This is what most people will be using it with.

However, PocketBeagle 2 is a pretty compact SBC, which can work well in cases where running full Linux might not be required. For such use cases, supporting an RTOS such as [Zephyr](https://www.zephyrproject.org/) can provide a great way to get the most performance out of the hardware.

So, continuing the work from M4F, I have created a [PR](https://github.com/zephyrproject-rtos/zephyr/pull/89130) to add Zephyr support for A53s. Currently, only revision A0 is supported.

The current setup of running Zephyr involves loading Zephyr from U-boot, similar to how the normal Linux images work. However, since Zephyr does not need the Linux rootfs, I have added a new [repository](https://github.com/beagleboard/bb-zephyr-images) to build SD card images for Zephyr. These images are a FAT32 boot partition with u-boot, tiboot3, tispl, and custom extlinux.conf. Initially, I was planning to use [debos](https://github.com/go-debos/debos) to build these images. However, I was still getting the same problem as described in the [issue](https://github.com/go-debos/debos). So, I switched to using mtools-based image building, which works completely in userspace (and thus inside docker).

The initial support only includes support for UART (on the debug port), but I will slowly add support for more peripherals. The goal is to have a [MicroPython](https://micropython.org/) image for PocketBeagle 2.

# Zephyr SD Card image support in BeagleBoard Rust Imager

Since I am now building Zephyr SD Card images for PocketBeagle 2, I have added support for these images to [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs). They are the same as Linux SD Card images but just don't have extra customization options.

The full changeset can be found in the [PR](https://github.com/beagleboard/bb-imager-rs/pull/29).

# Rust overlay abstractions patch

I have been working on adding Beagle Cape support for PocketBeagle 2, using a combination of export-symbols, i2c-extension, and other proposals. I decided to write the driver in Rust, which meant I had to also write a bunch of missing abstractions. Since the initial work on the driver is done, I have started the process of upstreaming the abstractions.

The first abstraction on the list is [overlay abstractions](https://lore.kernel.org/r/20250422-rust-overlay-abs-v1-0-85779c1b853d@beagleboard.or), which allows applying and removing fdt overlays from Rust. I will also be submitting other abstractions (such as sysfs, platform populate, etc) soon.

# Rust std r-efi update

The [PR](https://github.com/r-efi/r-efi-alloc/pull/8) to bump r-efi version in r-efi-alloc has been merged now. So r-efi can now finally be updated to r-efi 5.2.0 in the Rust std.

Check out the [PR](https://github.com/rust-lang/rust/pull/138737) for the full changeset.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs)
- [Rust std for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
- [Beagle Cape driver Tree](https://github.com/Ayush1325/linux/tree/b4/beagle-cape)
