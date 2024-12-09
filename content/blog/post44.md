+++
title = "My Week in Code #13"
description = "Updates regarding MicroBlocks, Devicetree Compiler, BeagleBoard Rust Imager and Arch Linux"
date = "2024-12-08T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "arch-linux", "microblocks"]
+++

Hello everyone. I was busy with the Linux kernel for most of the week. Let's go over everything.

# MicroBlocks Updates

MicroBlocks dev branch recently broke the [MicroBlocks Zephyr Port](https://openbeagle.org/beagleboard/microblocks). I sent a patch to the Discord channel, and it has been fixed.

# Devicetree Compiler

A lot of work this week was focused on the device tree compiler. I have been working on adding support for path references in devicetree overlays. In the initial patch, some parts of the initial patch series can be sent as independent patches. So last week mostly involved getting those merged.

## setprop namelen variants

Previously, all the setprop variants calculated length of property length from `strlen`. However, devicetree `__fixups__` contains information regarding the position of a property to resolve, and thus cannot use these normal variants. Additionally, it is not allowed to use heap memory in many places in the devicetree compiler and related tools, and thus, there was a need for `setprop_namelen_*` variants of functions similar to how these exist for `getprop_namelen_*`.

The [patch series](https://lore.kernel.org/devicetree-compiler/Z1KQJZpgODVajgd6@zatzit/T/#m8087a8818b06e73546b4f63cd683cbcdc512e16d) has been merged now.

## Clang format config

While working on the devicetree compiler code, I was using the clang-format config from the Linux kernel. As suggested by David Gibson, I sent a [patch](https://lore.kernel.org/devicetree-compiler/Z1KGnQt5W9XK1Xv2@zatzit/T/#m60c3078bf79053c05da0e19a5d37a2dcb1c95ad3) to add the clang-format config which has now been merged.

## Path Reference support in Devicetree overlays

Since the related patch for [setprop namelen](@/blog/post44.md#setprop-namelen-variants) variants has been merged, I have been working on version 2 of the path reference support patch. Since it is not allowed to use dynamic allocation, I have been able to get things working with adjusting offsets while growing. I still need to fix some tests and add new ones, but I should be able to post the patches this week.

# BeagleBoard Rust Imager

As per the suggestion in the [issue](https://openbeagle.org/ayush1325/bb-imager-rs/-/issues/41), I have been experimenting with moving around the configuration screen flow. A prototype can be found [here](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/36). I have not yet finalized whether this should be merged.

# Arch Linux

While working with docker containers, I found out that [systemd-networkd](https://wiki.archlinux.org/title/Systemd-networkd) is enabled by default, which conflicts with [NetworkManager](https://wiki.archlinux.org/title/NetworkManager). I did consider just using systemd-networkd and removing NetworkManager, but it seems NetworkManager is better for laptops. Additionally, I did not find good GUIs for `systemd-networkd`, so I have deactivated it for now.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [MicroBlocks Zephyr Port](https://openbeagle.org/beagleboard/microblocks)
- [Devicetree compiler mailing list](https://lore.kernel.org/devicetree-compiler/)
- [Arch Linux](https://archlinux.org/)
