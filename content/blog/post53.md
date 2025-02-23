+++
title = "My Week in Code #22"
description = "Updates regarding PocketBeagle 2 examples, devicetree, Rust std for UEFI and Zola flatpak"
date = "2025-02-23T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "pocketbeagle2", "rust", "uefi"]
+++

Hello everyone. This was a light week since I had college exams. Let's go over everything.

# PocketBeagle 2 Examples

I have been working on Rust examples for PocketBeagle 2 as evident from my past posts. However, a lot of examples ended up being more complicated than they should be, specially for new users. Additionally, Rust being a compiled language might not be suitable for fast prototyping. Maybe [cranelift](https://cranelift.dev/), can solve this problem, but I have not explored it enough to say yet. Additionally, it seems to require nightly, which I do not want to impose right now.

So, following the industry, I have also started adding examples in [Python](https://www.python.org/). Being probably one of the most popular scripting language, it seemed like the safe choice.

Additionally, I have started working on `beagle_linux_sdk` libraries for both Rust and Python. They are inspired by [gpiozero](https://gpiozero.readthedocs.io/en/latest/) for Raspberry Pi. Since `beagle_linux_sdk` is intentionally designed to be as high level and easy to use as possible, it allows making both the Rust and Python examples a lot more simple and readable. The plan is to push the libraries to [PYPI](https://pypi.org/) and [crates.io](https://crates.io/) once things are bit more stable.

As of writing this post, the updated examples are still in open [PR](https://openbeagle.org/beagleboard/vsx-examples/-/merge_requests/8). But they should be merged quite soon.

# Dynamic Devicetree overlays from userspace

While working on PocketBeagle 2 examples, I have started to realize how useful userspace application for devicetree overlays can be for single board computers. Some of the z that I envision are as follows:

1. **Dynamic Pin muxing:** A lot of SBC's aimed for creating hardware projects expose headers, where each pin can be used for multiple things like GPIO, I2C, PWM, etc, depending on the pinmux. I think Raspberry Pi has it's own solution to do userspace pinmux, but if userspace devicetree application was a thing, it could probably be used for this. Additionally, being able to use dynamic devicetree overlays for pin muxing would allow much easier transition to use proper device trees during production. 

2. **Dynamic Sensors/Devices:** Using devices such as sensors, external ADCs, EEPROMs, etc are also a common usecase in SBC's. A lot of current solutions seem to be designed around using user-space drivers in such cases. This is a bit of a shame since Linux kernel already has drivers for a lot of these drivers, and they are probably going to be of higher quality than most user space drivers.

There have been attempts in the past([OF: DT-Overlay configfs interface (v3)](https://lore.kernel.org/all/1417605808-23327-1-git-send-email-pantelis.antoniou@konsulko.com/#t) and [of/overlay: sysfs based ABI for dt overlays](https://lore.kernel.org/all/20161220190455.25115-1-xypron.glpk@gmx.de/)), but it seems like they never went anywhere.

I have started a [discussion](https://lore.kernel.org/all/9c326bb7-e09a-4c21-944f-006b3fad1870@beagleboard.org/) to see if I can find out what the reasons for the rejection were. Maybe things have changed by now, since the last patch series seems to be from 2017.

# Rust std for UEFI

## DevicePathNode Abstractions

In the [fs::exists](https://github.com/rust-lang/rust/pull/135368#discussion_r1947926136), I was asked to move the DevicePathNode abstractions to a seperate PR for easier review. It took a while due to my limited bandwidth, but I have created the [PR](https://github.com/rust-lang/rust/pull/137424) now. Hopefully, it will be merged soon, so that the initial `fs` support can finally land.

## Efi Service Bindings Abstractions

I have also started work on upstreaming networking with the [net stubs PR](https://github.com/rust-lang/rust/pull/136615). Continuing that, I have created [PR](https://github.com/rust-lang/rust/pull/137477) to add EFI service bindings abstractions.

UEFI Protocols such as TCP4/6, UDP4/6, etc use the EFI Service Binding protocol to create and destroy new instances. More information regarding this can be found in [UEFI Specification](https://uefi.org/specs/UEFI/2.11/11_Protocols_UEFI_Driver_Model.html#efi-service-binding-protocol).

I will try to keep all PRs as short as possible to make the process of upstreaming networking support move faster.

# Zola flatpak update

I am one of the maintainers of [Zola Flatpak Package](https://github.com/flathub/org.getzola.zola). Since it seems like everyone else was busy, I ended up updating the flatpak to [v0.20.0](https://github.com/flathub/org.getzola.zola/pull/14) that was released recently.

I also found that the metainfo upstream was out of date. So I also created [PR](https://github.com/getzola/zola/pull/2809) upstream to fix it.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [PocketBeagle 2 Examples](https://openbeagle.org/beagleboard/vsx-examples)
- [DevicePathNode Abstractions](https://github.com/rust-lang/rust/pull/137424)
- [EFI Service binding abstractions](https://github.com/rust-lang/rust/pull/137477)
- [UEFI Specification](https://uefi.org/specs/UEFI/2.11/11_Protocols_UEFI_Driver_Model.html#efi-service-binding-protocol)
- [Zola Flatpak](https://github.com/flathub/org.getzola.zola)
