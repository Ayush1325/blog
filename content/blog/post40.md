+++
title = "My Week in Code #9"
description = "Updates regarding BeagleConnect Freedom, BeagleBoard Rust Imager and Linux kernel"
date = "2024-11-10T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "beagleconnect-freedom", "beagleplay", "linux", "rust", "uefi"]
+++

Hello everyone. Another light week here. Let's go over everything.

# BeagleBoard Rust Imager

Continuing the trend from previous weeks, many developments took place in [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs).

## MacOS

Thanks to help from [Zain](https://openbeagle.org/superchamp234), BeagleBoard Rust Imager SD card flashing finally works on MacOS. With this, it is now possible to flash Linux images to an SD card on MacOS.

Any developers requiring similar functionality should look at the [PR](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/26). Here are the steps in brief:

1. Unmount the sd card using `diskutil`.
2. Create pipes using [`std::os::unix::net::UnixStream::pair()`](https://doc.rust-lang.org/src/std/os/unix/net/stream.rs.html#146-149).
3. Create a MacOS Authorization external form using [`security_framework::authorization`](https://docs.rs/security-framework/latest/security_framework/authorization/index.html).
4. Launch `authopen` with the `-extauth` argument using [`std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html) with the send pipe as stdout.
5. Send the MacOS authorization external form to the process stdin.
6. Read the `SCM_RIGHTS` control message from the receive pipe and get the file descriptor.

## Sd card formatting

SD card formatting is now supported on both Windows and Linux. MacOS support is still pending. This is useful for making the bootable SD cards usable again.

## Offline support

Offline use of BeagleBoard Rust imager for all supported boards is now possible. The base configuration now includes definitions for all boards.

# MicroBlocks

Since MicroBlocks IDE v2 has been [released](http://microblocks.fun/blog/2024-10-19-microblocks-2.0-launch/), there has been a new release of MicroBlocks stable firmware. You can grab the latest firmware for BeagleBoard Freedom from [here](https://openbeagle.org/beagleboard/microblocks/-/releases/250).

# GPIO Nexus Node Schema

While working on MikroBUS patches, I encountered [devicetree nexus nodes](https://devicetree-specification.readthedocs.io/en/v0.3/devicetree-basics.html#nexus-nodes-and-specifier-mapping). It allows the ability to create proxy GPIO controller thereby providing a stable numbering scheme for GPIO pins on MikroBUS socket.

To allow use in the Linux kernel, I created a [PR](https://github.com/devicetree-org/dt-schema/pull/143) to add the dt schema to the upstream repository. It has now been merged.

# Dynamic alias in Linux devicetree

Currently, adding/removing/updating `/aliases` is not supported in the upstream Linux kernel. This means overlays cannot rely on using `/aliases`, which are much more powerful than existing `/__symbols__`. In my [patch series](https://lore.kernel.org/all/20240902-symbol-phandle-v1-0-683efb2a944b@beagleboard.org/) to add support for phandles in `/__symbols__`, it was brought up that it might be better to switch to using `/aliases` instead of trying to add more to `/__symbols__`.

So I have created a [new patch series](https://lore.kernel.org/all/20241110-of-alias-v2-0-16da9844a93e@beagleboard.org/T/#t) based on the [original patch series](https://lore.kernel.org/all/1435675876-2159-1-git-send-email-geert+renesas@glider.be/) by Geert Uytterhoeven in 2015.

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom)
- [MicroBlocks Zephyr Port](https://openbeagle.org/beagleboard/microblocks/)
