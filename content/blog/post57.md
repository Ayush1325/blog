+++
title = "My Week in Code #26"
description = "Updates regarding BeagleBoard Rust Imager, devicetree specification and Rust std for UEFI"
date = "2025-03-23T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "devicetree", "uefi"]
+++

Hello everyone. A typical work week. Let's go over everything.

# BeagleBoard Rust Imager

I am gearing up for a v1 release of [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs) soon. As such, a lot of work went into it this week.

## BB-Downloader

While working on [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs), I initially tried searching for crates that provided the kind of downloading functionality I needed. The features I needed were the following:

1. Cache downloads to a specific location.
2. Use SHA256 for caching.
3. Live progress
4. Async

I did not find something that met all the criteria, so I ended up rolling my own. Initially, it was just part of `bb-imager`, but it was overdue for a refactor, so I thought might as well spin it up as its own crate. It is now in [crates.io](https://crates.io/crates/bb-downloader).

## BB-Config

BeagleBoard.org maintains a [distros.json](https://www.beagleboard.org/distros.json) which contains information regarding the distro images available for each board. The format is defined by Raspberry Pi Imager (old bb-imager was a fork of it), and is still being used by the Rust imager.

I have created a data structure (using serde) to parse and generate this config format. It was initially part of `bb-imager`, but since it was also in need of refactor, I have spun it off as an independent [crate](https://crates.io/crates/bb-config).

**Note:** While the config format used by `bb-imager-rs` is compatible with the old `bb-imager`, it is strictly speaking more of a subset. So it should probably not be used for working with the old imager.

## BB-Flasher

With all the refactoring of `bb-imager` library, the only functionality that was left in the crate was for flashing. So, I have renamed the crate to `bb-flaser` and just made a lot of general improvements in the code. Additionally, [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) and MSP430 support is now optional features. This is nice since that functionality does not work on macOS right now.

Some of the individual flashers ([bb-flasher-bcf](https://crates.io/crates/bb-flasher-bcf) and [bb-flasher-sd](https://crates.io/crates/bb-flasher-sd)) have already been pushed to crates.io. These aim to provide lower level functionality in a simple manner.

The `bb-flasher` crate acts as a wrapper over the lower level crates, while adding the following stuff:

1. Support for parsing `xz` compressed files.
2. Providing common trait based interface over different flashers.
3. Providing common trait based interface over different types of Os Images.
4. Async adapters

I have not pushed `bb-flaser` to crates.io yet since [PocketBeagle 2](https://www.beagleboard.org/boards/pocketbeagle-2) MSPM0 flasher is still not ready to be pushed to crates.io. Let's see when the time comes for it.

## GitHub Migration

As an experiment to attract more contributors, I have decided to migrate [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs) to GitHub. 

I am still in the middle of figuring out the migration to GitHub Actions, but things seem to be working decently well. GitHub CI has a lot of friendly actions, so it has been pretty simple. However, I feel like GitLab CI is still much easier to test locally, since there are no magic actions.

I also now have a [continuous integration pre-release](https://github.com/beagleboard/bb-imager-rs/releases/tag/continuous-release), which is quite useful. I also ended up simplifying the CI quite a bit (no longer building packages in PR CLIs) to avoid going over the free CI limits.

Let's hope that GitHub migration proves fruitful.

## Cargo Packager

As a part of GitHub migration, I ended up revisiting how I was generating the different packages. I was aware of the existence of [cargo-packager](https://docs.crabnebula.dev/packager/) for a while, but I had been bitten by [cargo-dist](https://github.com/axodotdev/cargo-dist) in the past, so I was a bit cautious regarding switching to something generic again.

However, since I was doing a huge migration anyway, I thought I might as well use something standard for packaging. I was pleasantly surprised to see how simply I was able to do migration for the GUI. Here are the things I like about it:

1. Much simpler that my custom Makefile/justfile solution.
2. Does not need me to create the deb package inside the cross container.
3. AppImage creation works flawlessly as well.
4. Now have a Windows installer using Wix.

However, I am not currently using it for the CLI. The reason is that there is no way to disable generating desktop entry right now. CLIs like bb-imager-cli should not have a desktop entry in the package, so this poses a problem. I have created a [PR](https://github.com/crabnebula-dev/cargo-packager/pull/313) to disable it, so hopefully, I can complete my migration soon.

I also think that it should probably migrate to the newer [go-appimage](https://github.com/probonopd/go-appimage) tools for generating AppImages (it currently uses linuxdeployqt). Hopefully, I will get time to create PR regarding this soon.

# Export Symbols Patch

Since there was almost no discussion in the v1 Export Symbols Patch, I have sent v2 with better examples, and trailers. Hopefully, I will get some feedback this time. Feel free to check it out in the [mailing list](https://lore.kernel.org/r/20250323-export-symbols-v2-1-f0ae1748b244@beagleboard.org).

# Rust std for UEFI

Some more work went into Rust std for UEFI this week.

## Implement some basics in UEFI fs

I have created a [PR](https://github.com/rust-lang/rust/pull/138662) to add some basic `std::fs` for UEFI:

1. Adds fs::canonicalize. Should be same as absolute in case of UEFI since there is no symlink support and absolute path is guaranteed to be uniqe according to spec.
2. Make fs::lstat same as fs::stat. Should be same since UEFI does not have symlink support.
3. Implement OptionOptions.

## mkdir

I have created a [PR](https://github.com/rust-lang/rust/pull/138667) to add support for `std::fs::create_directory`. Just a small self-contained part for `std::fs` since I have been trying to keep the PRs small enough to review.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs)
- [Export Symbols Patch v2](https://lore.kernel.org/r/20250323-export-symbols-v2-1-f0ae1748b244@beagleboard.org)
- [Rust std for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
