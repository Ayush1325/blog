+++
title = "My Week in Code #20"
description = "Updates regarding PocketBeagle 2, image building and Rust std for UEFI"
date = "2025-02-09T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "pocketbeagle2", "uefi"]
+++

Hello everyone. I ended up experimenting more than getting results, so not much to write. Let's go over everything.

# PocketBeagle 2 Examples

Continuing the work from [last week](@/blog/post50.md), more PocketBeagle 2 examples have been added.

Feel free to create [issues](https://openbeagle.org/beagleboard/vsx-examples/-/issues) regarding the examples you would like to be added.

## Button

Simple example to detect button presses. Uses Pin `P1_20` by default.

### Usage

```sh
debian@pocketbeagle2:~/vsx-examples/PocketBeagle2$ cargo run -p button
   Compiling button v0.1.0 (/home/debian/vsx-examples/PocketBeagle2/button)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 3.05s
     Running `target/debug/button`
Chip: gpiochip2 [600000.gpio] (92 lines), offset: 50
Button pressed
Button pressed
Button pressed
Button pressed
```

## Fade

Simple example to fade an LED in and out using PWM. Also contains abstractions to use PWM using [sysfs](https://docs.kernel.org/driver-api/pwm.html#using-pwms-with-the-sysfs-interface).

Uses `P1.36`. In case of [TechLab Cape](https://www.beagleboard.org/boards/techlab), this PIN is connected to `RGB.B`

### Usage

```sh
debian@pocketbeagle2:~/vsx-examples/PocketBeagle2$ cargo run -p fade -r
   Compiling fade v0.1.0 (/home/debian/vsx-examples/PocketBeagle2/fade)
    Finished `release` profile [optimized] target(s) in 3.85s
     Running `target/release/fade`
```

# BeagleBoard Image Builder

Currently, the image build process for all BeagleBoard images involves a [collection of scripts](https://openbeagle.org/beagleboard/image-builder) singlehandedly maintained by [Robert Nelson](https://openbeagle.org/RobertCNelson). Thanks to his amazing work on both handling the management of [openbeagle](https://openbeagle.org/) runners and image building, things have been working relatively well.

However, this also means that all the burden of pushing and maintaining new images falls on one person. Additionally, the images cannot currently be built in [openbeagle](https://openbeagle.org/), which limits the experimentation and having very special purpose images.

To hopefully improve the current situation, I have been working on trying to get image build working on openbeagle CI. I initially started with trying to get [debootstrap](https://wiki.debian.org/Debootstrap) directly working in CI, but while I could get it somewhat working (defining working very loosely here), it failed in CI.

Currently, I have 2 possible solutions I am exploring:
1. [debos](https://github.com/go-debos/debos/): Seems quite promising. Has good performance, and should work fine in CI. Additionally, it also can work with just user-mode-linux, so quite portable. Moreover, it already has a recipe system pretty similar to what I want in the final solution.
2. Using docker: This was mostly inspired from [yasib](https://openbeagle.org/jkridner/yasib). I was able to build rootfs for the most part, but there were quite a few warnings for systemlevel packages like systemd.

I am still experimenting with both of the above approaches. However, I have started leaning more towards debos. It can have much better performance than docker in docker, and already has been tested quite a bit. Additionally, I can always add support for 2nd approach to debos in the future.

Hopefully, I will have working images by next week.

# Optimize linux-embedded-hal dependencies

I have been using [linux-embedded-hal](https://github.com/rust-embedded/linux-embedded-hal) in some of the PocketBeagle 2 GPIO examples. When working with it, I observed that it was pulling some unnecessary dependencies even when `default-features` were disabled. I created a [PR](https://github.com/rust-embedded/linux-embedded-hal/pull/118) to fix it, which has now been merged.

# Rust UEFI Std Networking stubs

I was a bit bored, so I created the [PR](https://github.com/rust-lang/rust/pull/136615) to add the Networking stubs for UEFI. This will make the future networking PRs smaller and easier to review.

Hopefully, I will get around to working on it soon.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [Pocketbeagle 2](https://www.beagleboard.org/boards/pocketbeagle-2)
- [PocketBeagle 2 Examples](https://openbeagle.org/beagleboard/vsx-examples)
