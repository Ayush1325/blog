+++
title = "My Week in Code #33"
description = "Updates regarding PocketBeagle 2, BeagleConnect Freedom, Linux kernel and BeagleBoard Rust Imager"
date = "2025-06-01T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "pocketbeagle2", "beagleplay", "uefi", "zephyr"]
+++

Hello everyone. I ended up skipping some weekly updates since there was not much to talk about. Maybe I should make this series a bi-weekly thing instead of weekly updates. Anyway, I will be going over things from all the missed weeks as well. Let's get started.

# Zephyr

## Add MAIN domain USARTs for PocketBeagle 2 M4F

The initial support for [PocketBeagle 2 M4F](https://docs.zephyrproject.org/latest/boards/beagle/pocketbeagle_2/doc/index.html) landed a while ago. This support was quite limited, and is being expanded with time. Initially, I thought that the M4F, which is from MCU domain, did not have access to the MAIN domain peripherals. However, this assumption turned out to be wrong.

The cross-domain peripheral support varies a bit, but for the most part, it is possible to use MAIN domain peripherals from MCU domain. This meant that more of the PocketBeagle 2 header functionality could be used from the M4F core. However, the current base devicetree for AM62x M4F did not define nodes for the MAIN domain peripherals.

Since the initial support for M4F was limited to MCU domain UARTs, I started by adding support for MAIN domain USARTs. It is important to note that the M4F does not have access to the interrupts for the MAIN domain USARTs. Thus, only polling based usage is supported.

I have also started following the Linux kernel devicetree nomenclature of having the domain prefix in the peripherals. I will rename the already existing peripheral nodes in the future.

Checkout the full [PR](https://github.com/zephyrproject-rtos/zephyr/pull/90456) for more details.

## Add I2C support for PocketBeagle 2 A53

The initial support for [PocketBeagle 2 A53s](https://docs.zephyrproject.org/latest/boards/beagle/pocketbeagle_2/doc/index.html) was merged recently. This allows running ZephyrRTOS on the main A53 cores instead of Linux. It uses the same u-boot as Linux, but can be great in very special purpose and high performance real-time use cases.

The initial support was limited to just USARTs. However, I have created a PR to enable I2C pins on the header. Checkout the [PR](https://github.com/zephyrproject-rtos/zephyr/pull/90637) for more details.

# BeagleBoard Imaging Utility

## macOS signing

For the longest time, the BeagleBoard imaging utility was not signed on any platform. This was a big problem in the case of macOS since without proper signing and notarization, it can be quite confusing and difficult for non-tech savvy people to get it to launch. Since I do not own a mac, it was also quite difficult to test, and thus, I have been mostly ignoring it.

However, with the help from [Deepak](https://github.com/lorforlinux), I was able to get the macOS packages properly signed and notarized. So, hopefully, this will help attract more users, which in turn might become contributors.

The first release with proper signing is [v0.0.6](https://github.com/beagleboard/bb-imager-rs/releases/tag/v0.0.6).

## Ubuntu 22.04 support

Since switching the development to GitHub, I have been using ubuntu-24.04 runner for Linux builds. However, it was recently brought to my attention that this makes the package not work on older systems such as Ubuntu 22.04, due to glibc. For now, I have switched to building on ubuntu-22.04 runners. However, maybe it would be a good idea to either use Debian docker containers to build the image, or use musl rust targets and do static linking of libc.

## GPT partition table support

Most of the BeagleBoard Linux images used the MBR partition table, so GPT partition table was not supported in the BeagleBoard Rust Imager. However, it recently came to my attention that the [BeagleV-Fire](https://www.beagleboard.org/boards/beaglev-fire) images use the GPT partition table, and thus was failing. However, I have now added initial support for GPT, so hopefully everything works fine (I have only done Linux testing).

Checkout the [PR](https://github.com/beagleboard/bb-imager-rs/pull/38) for more details.

## USB DHCP Option

Thanks to the amazing work by [Robert](https://github.com/RobertCNelson), network sharing over USB works great on Windows and macOS. However, it might not always be desirable to have USB NCM support. Thus, it is not possible to enable or disable this from both the CLI and GUI.

Checkout the [PR](https://github.com/beagleboard/bb-imager-rs/pull/34) for more details.

## Refresh button for Image list

BeagleBoard Imager downloads the list of images and firmwares at the start of each run. This sometimes can go wrong because network requests can fail. Earlier, the only real way to fix this was to relaunch the application, after ensuring a working network. However, I have now added a refresh button in the image selection page that allows manually refreshing the list.

This ended up being a bit more complicated than it might first appear since the distro list is not a single remote list. Instead, it is a result of merging multiple seperate lists. So the refresh button only appears at the first image selection page (no sub-pages).

Checkout the [PR](https://github.com/beagleboard/bb-imager-rs/pull/35) for more details.

## SSH Auth key support

BeagleBoard Imager now supports specifying authorized ssh keys in the UI, similar to Raspberry Pi imager. Most people using ssh will love this option.

Checkout the [PR](https://github.com/beagleboard/bb-imager-rs/pull/32) for full details.

# BeaglePlay CC1325P7 Micropython support

Unofficial builds for Micropython on BeaglePlay CC1325P7 have existed for a while. However, since the version of Zephyr in upstream Micropython has been updated, I have created an upstream PR to add support for this MCU. Hopefully, people will find it useful.

Checkout the [PR](https://github.com/micropython/micropython/pull/17274) for more details.

# Rust Standard Library for UEFI

## TCP4 write support

Recently, the initial support for TCP4 (which just included connect support) was merged in upstream Rust. Following on the initial PR, I have created a [PR](https://github.com/rust-lang/rust/pull/141532) that implements support to send data over the TCP4 connection. Let's hope that it gets merged soon.

## Random generation fallback

It was reported in a github issue that some UEFI firmwares do not support the EFI_RNG_PROTOCOL. This causes HashMaps to fail on such systems. I have created a [PR](https://github.com/rust-lang/rust/pull/141324) to fall back to rdrand on x86 and x86_64 systems if the protocol is missing. This seems to work for the tested systems. I will try adding a similar fallback for ARM systems if someone reports a similar problem.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs)
- [Rust STD for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
