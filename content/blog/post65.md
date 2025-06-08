+++
title = "My Week in Code #34"
description = "Updates regarding PocketBeagle 2, BeagleConnect Freedom, Linux kernel and BeagleBoard Rust Imager"
date = "2025-06-08T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "pocketbeagle2", "zephyr", "armbian", "microblocks"]
+++

Hello everyone. A typical work week. Let's go over everything.

# Fix MicroBlocks CI

Recently, the MicroBlocks CI which builds microblocks dev and pilot branches have been failing. After a bit of testing, I found that a `cmsis_6` zephyr module is now required by [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom). The CI was fixed by allowing the module in the custom `west.yml`.

Checkout the [patch](https://github.com/beagleboard/microblocks-zephyr/commit/5e6c39362c96b0f539d94a942b31d46ec0c68382) for more details.

# BeagleBoard Rust Imager

## Improve Windows Performance

For a while, the Windows implementation of flashing images has been different from Linux and macOS. On Linux and macOS, customization was performed after writing the image. However, on Windows, I was not able to open the `sysconf.txt` file for some reason. So on Windows, I was using the following hack:

1. Create a temporary copy of the image, or extract to a temporary copy (if the image is compressed).
2. Perform customization on this temporary file.
3. Write the modified file to the SD Card.

One of the major reasons for not fixing the problem was that I did not have a Windows machine on hand, and my previous machine was too slow to do things in VM. Recently, I got my hands on an old laptop, which while not fast, is ok enough for testing. So after testing various things, and going through the Raspberry Pi imager source, I was able to conclude that Windows does not like non-aligned reads/write on SD Card. Taking inspiration from the Raspberry Pi imager, I created a wrapper that always accesses data in blocks of 4096 bytes. This seems to fix the Windows issue, allowing removing the need for a temporary working copy. Hopefully, it will provide a noticeable speedup in windows flashing.

Check out the full [PR](https://github.com/beagleboard/bb-imager-rs/pull/39) for more details.

## Disable customization for local images

Recently, it was pointed out that some of the old Beagle images do not have support for sysconf.txt. Additionally, it can be confusing for new users to have customization options present in local images, since there is nothing to check if customization is supported on such images. So, I have disabled customization options for all local images for now. I am planning to add a toggle to allow developers to apply customization options on local images, but it should be labeled as experimental since it can lead to flashing errors.

Check out the full [PR](https://github.com/beagleboard/bb-imager-rs/pull/43) for more details.

# Enable PocketBeagle 2 A53 GPIOs

The GPIO subsystem right now has a limitation of only supporting 32 pins per port. While this is a problem that needs to be fixed, all the built-in LEDs in PocketBeagle 2 use pins that are in the supported range. So I have enabled the LEDs for now. This should also help in future PRs that make changes to the GPIO subsystem since PocketBeagle 2 will be used as the testbed.

Check out the full [PR](https://github.com/zephyrproject-rtos/zephyr/pull/91051) for more details.

# Previous Property support in devicetree compiler

Since I got some feedback on the v2 of previous-property patches, I have posted [Patch v3](https://lore.kernel.org/r/20250605-previous-value-v3-0-0983d0733a07@beagleboard.org) to address the comments. This patch series allows constructing devicetree property from previously defined values, enabling things like append, pre-append, etc.

# Fix Armbian config overwrite

There has been a lot of work to get [Armbian](https://www.armbian.com/) working on BeagleBoard boards. Recently, it was discovered that the Armbian kernel had ext4 support built as a module instead of being built-in. This was a problem since the rootfs is ext4. After a bit of testing, I found that the Armbian build script was overwriting the config for docker, and converting the ext4 support from built-in to a module.

To fix this, I have created a [PR](https://github.com/armbian/build/pull/8276) that ensures that ext4 (and other drivers) are built as modules only if they are disabled. In case they are set as built-in by some prior config, they are not touched.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs)
- [MicroBlocks](https://github.com/beagleboard/microblocks-zephyr)
