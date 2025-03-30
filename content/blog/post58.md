+++
title = "My Week in Code #27"
description = "Updates regarding BeagleBoard Rust Imager and Rust std for UEFI"
date = "2025-03-30T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "uefi"]
+++

Hello everyone. A typical work week. Let's go over everything.

# BeagleBoard Rust Imager

## GitHub Migration

The migration of [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs) to GitHub has now been completed. I have also created [v0.0.4 release](https://github.com/beagleboard/bb-imager-rs/releases/tag/v0.0.4) which brings a lot of fixes and marks the end of the migration.

## Eject After flashing

In prior versions of BeagleBoard Rust Imager, I did not eject the SD card after flashing. This would cause weird issues on Linux like not being able to remount the SD card without removing it first. So, the flasher now ejects the SD card after flashing on Linux.

Windows and MacOS still do not eject after flashing.

## Windows Customization Fixing

Last week Maique Correa Garcia created an [issue](https://github.com/beagleboard/bb-imager-rs/issues/12) regarding flashing failure on Windows. On investigating it further, I found that the customization step (which sets username, hostname, etc) was causing the failure on Windows. 

I was able to figure out the initial error quickly. It seems that Windows does not seem to know the size of the SD card which does not have a valid partition table. This meant that when [mbrman](https://crates.io/crates/mbrman) crate was trying to guess the size by seeking to the end, it would fail on Windows. The solution to that was to only try reading the MBR partition table.

However, after fixing that, I quickly ran into another problem. To give some context, the flow of bb-imager SD card flashing is as follows:

1. Open the SD card as [std::fs::File](https://doc.rust-lang.org/std/fs/struct.File.html) in platform dependent manner.
2. Write the OS image to the SD Card File.
3. Read the MBR table from the SD Card File.
4. Open/create `sysconf.txt` in the FAT32 `BOOT` parition of the SD Card.
5. Append the entries to `sysconf.txt`.

For some reason, Windows would fail on step 4. In fact, I was not able to read anything from the `BOOT` partition on Windows, even though the same code worked fine on Linux.

I did try various things, and tried looking at RPI-imager for inspiration, but was not able to solve the problem. Maybe the problem is something with the way [fatfs](https://crates.io/crates/fatfs) works. 

I then moved onto alternative approaches like having a memory layer on top of the base image which would allow applying customizations in memory, before writing to an SD card. However, this turned out to be a dead end since that would mean needing to have `Seek` support in the Os Image file, which is not always possible (since the images can be compressed as XZ). I also did not want to create a temporary copy of the extracted image, since that is problematic on devices with limited storage (The flasher can be run on our boards as well).

In the end, I settled on having a separate flashing implementation for Windows, which creates a temporary copy of the extracted OS Image, applies customizations to this copy, and then writes the modified Os Image to an SD Card. I feel like if your machine is running something as heavy as Windows, it should probably be fine to use 8G of temporary file storage.

## Update Crates

I have pushed new versions of the following crates:

- [bb-downloader](https://crates.io/crates/bb-downloader)
- [bb-flasher-bcf](https://crates.io/crates/bb-flasher-bcf)
- [bb-flasher-sd](https://crates.io/crates/bb-flasher-sd)

## Isaac Parker's Contributions

I would like to thank Isaac Parker for his [contributions](https://github.com/beagleboard/bb-imager-rs/issues?q=is%3Apr+author%3Aparrotmac) last week. Hopefully, BeagleBoard Rust Imager will attract more external contributions with time.

## UI Code Improvement

I am using [iced](https://iced.rs/) as the GUI toolkit for BeagleBoard Rust Imager. It is a cross-platform GUI library for Rust, inspired by Elm. Since the UI code in iced is also just normal Rust (as opposed to [Slint](https://slint.dev/) which has a separate language for UI), it can be a bit daunting for new contributors, especially the ones who only want to contribute to the UI.

While I have no plans to change the toolkit right now, I decided to isolate the code to a single module, to hopefully make the code more readable and maintainable in the long run. Checkout the [PR](https://github.com/beagleboard/bb-imager-rs/pull/21) for more details.

# Implement File Times for UEFI

I have created a [PR](https://github.com/rust-lang/rust/pull/138918) to implement fs File Times for UEFI. This involves adding conversion methods from Unix Time to UEFI Time. I would like to thank everyone involved with the algorithms present [here](http://howardhinnant.github.io/date_algorithms.html).

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs)
- [Rust std for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
