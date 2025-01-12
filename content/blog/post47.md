+++
title = "My Week in Code #16"
description = "Updates regarding BeagleBoard Rust Imager, Devicetree compiler and Rust fs support for UEFI"
date = "2025-01-12T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "rust", "pocketbeagle2", "uefi", "linux", "c"]
+++

Hello everyone. A typical week for development. Let's go over everything.

# BeagleBoard Rust Imager

## Pocketbeagle2 MSPM0 support

[Pocketbeagle2](https://www.beagleboard.org/boards/pocketbeagle-2) uses MSPM0 as both EEPROM and ADC. The EEPROM is used to store board-specific information in all BeagleBoard boards for easier debugging and differentiation of different revisions. 

While the pb2_mspm0 driver supports updating firmware, it isn't suitable to be used directly by new people:
1. Does not persist EEPROM contents.
2. Does not support Ti-TXT directly.

To solve this, I have added pb2_mspm0 support directly to `bb-imager-cli`. There is also a flag to control if EEPROM contents should be preserved (since it is possible to flash firmware other than the default one). Additionally, since `bb-imager-rs` supports Ti-TXT and iHex by default, we get that support for free.

I am still trying to figure out how to add support to the GUI since the flashing requires access to `sysfs` entries, which are owned by root (and probably should stay that way). I think I am supposed to use `polkit`, but not completely sure.

## Contribution and Packaging Instructions

I have added instructions regarding [Contribution](https://openbeagle.org/ayush1325/bb-imager-rs/-/blob/main/CONTRIBUTING.md?ref_type=heads) and [Packaging](https://openbeagle.org/ayush1325/bb-imager-rs/-/blob/main/PACKAGING.md?ref_type=heads) to hopefully make it easier for newcomers to get started. It also helped be document a lot of stuff that even I forget, so it should prove useful.

## Better cross-compilation support

For pocketbeagle2 testing, I wanted to be able to cross-compile from my main machine without needing to tinker too much. I had used [cross](https://github.com/cross-rs/cross) in the past, but it turns out that the container cross uses by default is based on Ubuntu 20.04, which contains libssl1. Pocketbeagle2 images have libssl3, which means that I could not use the default images for cross-compiling.

I have added custom image dockerfiles, so cross-compiling should be as simple as setting `RUST_BUILDER=cross` and running a make recipe.

Note: cross compiling is only supported for the following targets right now:
- x86_64-unknown-linux-gnu
- aarch64-unknown-linux-gnu
- armv7-unknown-linux-gnueabihf

Windows should support can probably be added with a custom dockerfile (or the default cross windows container might work), but I do not need it much, so haven't really tried adding it.

# Export Symbols Devicetree Support

A few weeks ago, Herve Codina sent patches to the kernel mailing list for supporting [export-symbols](https://lore.kernel.org/all/20241209151830.95723-1-herve.codina@bootlin.com/) as another possible way to support addon board overlays. Since adding support for path references to overlays ended up being a dead end (see [here](https://lore.kernel.org/devicetree-compiler/6b2dba90-3c52-4933-88f3-b47f96dc7710@beagleboard.org/T/#m8259c8754f680b9da7b91f7b7dd89f10da91d8ed)), I have starting working on `export-symbols` based approach for mikroBUS.

As a part of that effort, I am working on adding `export-symbols` support to the base devicetree compiler, to hopefully make it part of devicetree specification instead of just a Linux specific thing. That way, it should be possible to support the same addon-board overlays in Zephyr in the future.

## Base devicetree compiler

I started work with adding support to the base devicetree compiler. The [patch series](https://lore.kernel.org/r/20250110-export-symbols-v1-1-b6213fcd6c82@beagleboard.org) does not contain tests right now since this is more of an RFC for now.

## fdtoverlay

Similarly, I also sent the [patch series](https://lore.kernel.org/r/20250112-export-symbols-overlay-v1-1-e94526d349ef@beagleboard.org) for support in fdtoverlay. The tests will be added to future versions of the patch.

## Future work

As discussed in the [original patch series](https://lore.kernel.org/all/20241209151830.95723-1-herve.codina@bootlin.com/), to make use with `fdtoverlay` possible, we need to add a way to pass the target path to `fdtoverlay`, which is not supported right now. I will try to work on it this week. Hopefully, addon board support will be merged before 2026.

# Rust fs support for UEFI

After some discussion in Zulip, I have broken up the original [std::fs PR](https://github.com/rust-lang/rust/pull/129700) into smaller chunks to make reviewing simpler.

## Unsupported fs support

The first [PR](https://github.com/rust-lang/rust/pull/135324) as a part of this series was just to add the `sys::pal::uefi::fs` module with everything left to `unsupported`. It has been merged now.

## Implement `fs::exists`

I have also created a [PR](https://github.com/rust-lang/rust/pull/135368) just to add `fs::exists` support since it allows testing the `File::open` algorithm without implementing other parts of File API. However, it was still quite big, so I split it into smaller PRs.

## OwnedDevicePath

Branching off from `fs::exists` [PR](https://github.com/rust-lang/rust/pull/135368), the [OwnedDevicePath PR](https://github.com/rust-lang/rust/pull/135393) adds support for the abstractions common to the already merged `std::process` module. This can hopefully reduce noise and make review easier from the `fs::exists` PR.

## is_absolute Refactoring

I am also planning to split the `absolute` path creation support present in `fs::exists` [PR](https://github.com/rust-lang/rust/pull/135368) into it's own PR implementing `std::path::absolute`. However, the current internal structure of `std::path` APIs cannot support UEFI style paths well (see [here](https://github.com/rust-lang/rust/issues/52331)). This causes a problem that any `absolute` path constructed for UEFI will return `false` for `std::fs::Path::is_absolute`, which can be quite confusing.

In an attempt to fix this, I have created a [PR](https://github.com/rust-lang/rust/pull/135405) to move the logic for `is_absolute` into `std::sys`, and make it platform dependent (which it already kind of was, but was pretending not to be). I'm not sure if this will get merged, though, so let's see.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs)
- [Linux kernel export-symbols patch series](https://lore.kernel.org/all/20241209151830.95723-1-herve.codina@bootlin.com/)
- [Devicetree Compiler export-symbols patch series](https://lore.kernel.org/r/20250110-export-symbols-v1-1-b6213fcd6c82@beagleboard.org)
- [Fdtoverlay export-symbols patch series](https://lore.kernel.org/r/20250112-export-symbols-overlay-v1-1-e94526d349ef@beagleboard.org)
- [Rust std for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
