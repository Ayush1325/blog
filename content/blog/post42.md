+++
title = "My Week in Code #11"
description = "Updates regarding BeagleBoard Rust Imager and UEFI Rust std"
date = "2024-11-24T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "uefi"]
+++

Hello everyone. Another light week here. Let's go over everything.

# BeagleY-AI SD Card support

While hacking on my BeagleY-AI, I found that my HP SD card did not work with BeagleY-AI for some reason. On digging a bit further, some SD cards were not working on BeagleY-AI due to some issues with signal voltage switching on the am62x platform. This has now been fixed, as seen in the following [patch series](https://lore.kernel.org/all/20240924195335.546900-1-jm@ti.com/). By switching out the Linux kernel to mainline, I was able to get the HP SD card to work. So, the SD card compatibility of BeagleY-AI will improve soon.

# BeagleBoard Rust Imager

BeagleBoard Rust imager got lots of love this week.

## AppimageLauncher Problem

As pointed out by Jason [here](https://openbeagle.org/ayush1325/bb-imager-rs/-/issues/37), the BeagleBoard Rust imager appimage is not usable with [AppimageLauncher](https://github.com/TheAssassin/AppImageLauncher). The issue seems to be that [appimagetool](https://github.com/AppImage/appimagetool) uses `ZSTD` compression now, while `AppimageLauncher` does not support it yet. Moreover, it seems like `AppimageLauncher` makes the appimage unusable. For now, the recommendation is to either remove `AppimageLauncher` or at least deactivate it for the BeagleBoard Rust imager.

More information regarding the upstream issue can be found [here](https://github.com/TheAssassin/AppImageLauncher/issues/674).

## SD Card Flashing Performance Improvements

While the SD card flashing worked, it was much slower than the original [bb-imager](https://openbeagle.org/beagleboard/bb-imager). After some tinkering with the buffer size while flashing and using parallel decompression of xz images, SD card flashing should be much faster (flashing time decreases by 57%).

It is still a bit slower than the original bb-imager (about 1 min), but it seems related to the internals of `QFile`, so that would be much more work than just a day.

## Debian Packages

BeagleBoard Rust imager now builds Debian packages in [CI](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/33) using [cargo-deb](https://github.com/mmstick/cargo-deb). While the goal is to eventually have an upstream package using [debcargo](https://github.com/ahdinosaur/debcargo), there is not much point until `bb-imager-rs` hits v1.0.0. Additionally, since debcargo only seems to support building stuff already present in [cartes.io](https://crates.io/), it does not seem suitable for experimental builds (although I could be wrong about that).

## Experimental Builds

Builds from the latest `main` branch are now pushed to [Package Registry](https://openbeagle.org/ayush1325/bb-imager-rs/-/packages) rather than just being stored as build artifacts. This makes grabbing the latest development version of `bb-imager-rs` much simpler.

Note: Only the most recent build is kept around.

# UEFI Process Args in Rust std

Initial support for Rust `std::process` was [merged](https://github.com/rust-lang/rust/pull/123196) upstream a while back. However, at the time, it was missing support for passing arguments to the spawned process to keep the PR easier to review. Finally, the support for arguments has now been merged to make the API somewhat feature-complete.

UEFI process is one of the more important APIs since it is used by Rust to spawn tests, thereby bringing us closer to being able to run all of Rust's testing infrastructure on UEFI. This will help make Rust much easier to use in the UEFI space.

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [My Rust std for UEFI open PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
