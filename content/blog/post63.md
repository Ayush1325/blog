+++
title = "My Week in Code #32"
description = "Updates regarding PocketBeagle 2, BeagleConnect Freedom, Linux kernel and BeagleBoard Rust Imager"
date = "2025-05-04T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "pocketbeagle2", "beagleconnect_freedom"]
+++

Hello everyone. A typical work week. Let's go over everything.

# Use extlinux in Armbian images

Thanks to all the amazing work by [Andrei Aldea](https://github.com/Grippy98), PocketBeagle 2 and other BeagleBoard.org boards support [Armbian](https://www.armbian.com/). By default, it seems Armbian uses the old uEnv.txt file to specify the boot entry. This is fine, but the newer [extlinux.conf](https://github.com/ARM-software/u-boot/blob/master/doc/README.distro) is much more powerful, and is already used in normal BeagleBoard.org Debian images.

There are a few reasons to use extlinux.conf:

1. First-class support in U-boot.
2. Allows multiple boot entries: Great when doing kernel development.
2. Allows fallback to option 1 in case of boot failure.

I have created a [PR](https://github.com/armbian/build/pull/8130) to use extlinux.conf in PocketBeagle 2 as a start. I will expand to other boards soon.

# Linux Kernel

I tried moving the needle forward regarding connector + addon-board setups upstream. Not much progress, but well, what else is new.

## Send of_platform_populate abstractions patch series

Since I have decided to write the Beagle cape driver in Rust, I was trying to upstream some of the abstractions I need. As a part of this work, I tried sending abstractions for `of_platform_populate` and `of_platform_depopulate` in the following [patch series](https://lore.kernel.org/r/20250429-rust-of-populate-v2-1-0ad329d121c5@beagleboard.org). However, it seems like that function is semi-deprecated. Additionally, it seems like it would be better to send a driver patch with all the abstractions before trying to send the abstractions individually since it is an untested driver.

So the upstreaming rust abstractions are probably on hold for a while. I guess if someone else wants to pick up the abstractions for their drivers, they are in my [tree](https://github.com/Ayush1325/linux/tree/b4/beagle-cape).

## Start discussion for connector + addon-board setups

I have mostly worked with the assumption that any connector + addon-board setup will need to only do manipulation of the tree inside the connector node, rather than the global tree. This was derived from some of the other discussions I have had in the past. However, it seems like not everyone agrees regarding that. So I have created a [discussion thread](https://lore.kernel.org/linux-devicetree/e05c315d-a907-45f0-8f5c-1c106b05f548@beagleboard.org/T/#m12c678c2eb0daea8629bbe35607434b29fb55ed5) to come to a consensus once and for all. I do not want to repeat the same discussion in every thread.

Hopefully, things get resolved soon one way or the other.

# BeagleConnect Freedom

## Enable watchdog timer in base devicetree

Since MicroPython now supports Zephyr v4.0.0, I was working on adding beagleplay/cc1352p7 support. However, it turns out, that BeagleConnect Freedom is currently broken in upstream MicroPython. This is due to a new requirement of MicroPython for a watchdog timer in devicetree. This was disabled in the upstream BeagleConnect Freedom devicetree. I have created a [PR](https://github.com/zephyrproject-rtos/zephyr/pull/89328) to enable it.

I should also probably create a weekly GitHub action to ensure that BeagleBoard boards do not break, similar to what I am doing for MicroBlocks. There are also plans for maintaining an upstream Zephyr tree specifically for BeagleBoard boards. Stay tuned for that.

## Enable ADC Support in MicroPython

I have also enabled ADC support in upstream MicroPython for BeagleConnect Freedom now. Here is the [PR](https://github.com/micropython/micropython/pull/17223).

# Improve init format handling in BeagleBoard Imaging Utility

[Last week](@/blog/post62.md), I introduced Zephyr SD Card images for PocketBeagle 2. These images are intended for running ZephyrRTOS on PocketBeagle 2 A53 cores instead of Linux and thus do not need rootfs. This also means that sysconf.txt based customization options are not useful for these images.

I added a hacky workaround for these images last week to get initial support in [BeagleBoard Imaging Utility](https://github.com/beagleboard/bb-imager-rs). But since I had some time, I have added proper support for specifying `init_format` in the config.json. This is the same as the Rpi config, so things are still backward compatible.

The changeset can be found in the [PR](https://github.com/beagleboard/bb-imager-rs/pull/31).

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs)
- [Beagle Cape driver Tree](https://github.com/Ayush1325/linux/tree/b4/beagle-cape)
- [Discussion for connector + addon-board setups](https://lore.kernel.org/linux-devicetree/e05c315d-a907-45f0-8f5c-1c106b05f548@beagleboard.org/T/#m12c678c2eb0daea8629bbe35607434b29fb55ed5)
