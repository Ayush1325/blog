+++
title = "My Week in Code #12"
description = "Updates regarding BeagleBoard Rust Imager and Arch Linux"
date = "2024-12-01T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "arch-linux"]
+++

Hello everyone. I switched my linux distro this week. So lots of updates regarding it. Let's go over everything.

# BeagleBoard Rust Imager

Since CI was broken last week, these changes have yet to be merged.

## Improve Cancel Messages

As pointed out by Deepak [here](https://openbeagle.org/ayush1325/bb-imager-rs/-/issues/40), the messages when the cancel button was used were not particularly informative regarding which step of the flashing process was canceled. This was due to the initial quick and dirty implementation for everything. This has now been fixed to provide better messages for each stage of flashing. Here are the messages:

- Preparation canceled by user
- Downloading canceled by the user
- Flashing canceled by user
- Verification canceled by user
- Customization canceled by the user

## Speedup SD card verification

After some testing for verification, I found out that a 4K buffer length seems to be the fastest for reading SD card contents for SHA256 verification. This gives a modest performance boost. However, verification is still not as fast as I would like. I may try vectored reads in the future.

## Appimage CI broken

Last week, [appimagetool](https://github.com/AppImage/appimagetool) got a new update that broke my CI. I have not yet found the time to figure out the issue, but let's see if I can reproduce it in a local docker. I have created an [issue](https://github.com/AppImage/appimagetool/issues/79) for the time being.

# ArchLinux Switch

I have been using [Fedora Sway Atomic](https://fedoraproject.org/atomic-desktops/sway/) since Fedora 36. It has been a good experience for the most part. I appreciated how rock-stable it was. I have even rebased to the KDE atomic variant in the past and was surprised at how painless it ended up being. Since I used Neovim, working inside containers was not particularly difficult either. All in all, it was a pretty great experience.

While I can come up with a lot of excuses regarding why I switched back to ArchLinux, the honest one was that I was bored. In the past few years, I have become much more comfortable contributing to random projects. Additionally, I have also gained a lot more knowledge regarding the general design of Linux systems, along with the basics of debugging. Along with the fact that I do a lot of Linux kernel development nowadays (most for embedded systems), I am okay with my system breaking.

The great thing about Fedora Atomic is that I could just switch to a previous deployment if something broke. However, it also meant that I had less incentive to go and create upstream issues and dive into the source to see if I could fix the problem myself. This was great when I was just getting started with open-source and did not have the time or knowledge to just try fixing anything broken I found. However, as someone who is a decent developer now, I have found that fixing problems in random stuff is actually quite a good way to improve my skills.

## What I like about Arch

### PKGBUILD

I love the Arch [PKGBUILD](https://wiki.archlinux.org/title/PKGBUILD) based system. While I like [AUR](https://wiki.archlinux.org/title/Arch_User_Repository), the real reason I love PKGBUILD is because how easy it allows me to manage things like custom local kernels and some other experimental software with pacman instead of having to remember where I put all the files.

This makes working on development builds much simpler since, well, even if I forget or lose the exact way I installed the package (which I do more than I should), I do not need to worry that I might have forgotten some man page files while manually removing the software.

### Arch User Repository

Almost every piece of software available on Linux can be installed on Arch Linux thanks to [AUR](https://wiki.archlinux.org/title/Arch_User_Repository). While the stability of AUR packages can be a bit questionable, if you can read PKGBUILD, it is much better than using [Fedora Copr](https://copr.fedorainfracloud.org/) or [Ubuntu PPAs](https://launchpad.net/ubuntu/+ppas). Additionally, AUR also provides simple access to git versions of packages, which is great in cases where some functionality is not yet available in the stable version of the software.

### Arch Wiki

I do not think I need to explain this point much. [Arch Wiki](https://wiki.archlinux.org/title/Main_page) is probably the best documentation for any Linux distribution. I have used it a lot, even when dealing with problems in Fedora.

## My Setup

I am using [SwayWM](https://wiki.archlinux.org/title/Sway) with [Wayar](https://github.com/Alexays/Waybar). I took a lot of applications and configs from [Fedora Sway Atomic](https://fedoraproject.org/atomic-desktops/sway/) since I think the defaults are pretty good on it. I also switched to [fuzzel](https://codeberg.org/dnkl/fuzzel) from rofi, and it seems enough for my needs since it does support dmenu style runner.

All my configs can be found [here](https://github.com/Ayush1325/dotfiles).

## Challenges

While the initial install was quite simple thanks to [archinstall](https://wiki.archlinux.org/title/Archinstall), there were some issues that I had to deal with. I probably will encounter more as time goes on.

### Systemd initial ramdisk

The default configuration generated by archinstall uses busybox-based initial ramdisk. While it works fine, I was intrigued by system-based init. I have heard that systemd initial ramdisk can be faster in a lot of cases, but I have not done any testing either way.

The [mkinitcpio page](https://wiki.archlinux.org/title/Mkinitcpio) documents the different hooks that are needed for systemd init. However, I was not aware that I needed to edit systemd entry to use systemd based decryption. This led me to have a broken initial ramdisk, which could not even drop me into some root shell (since the root still needed to be unlocked).

I was able to fix this by fixing the system-boot entry using an arch live iso as described [here](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Using_systemd-cryptsetup-generator).

### Locale Problems

During `archinstall`, I selected `en_IN` as my locale. It generated locales fine, but it is recommended to also generate `en_US.UTF-8` as a backup. While most things worked fine, I had weird problems with some software like [fuzzel](https://codeberg.org/dnkl/fuzzel), which did not like my `LANG` to be set to `en_IN` without specifying the encoding. I was able to fix this by setting `LANG=en_IN.UTF-8` in `/etc/locale.conf`.

### Other Steam problems

I am using the Steam package in normal Arch Linux repos instead of Flatpak. However, initially, it failed to launch. After looking at the logs, it turned out that Steam wanted a few directories to be present in the system: `~/.local/share/{icons, font}`.

I have also been facing some freeze problems when running games ([Persona 5 Royal](https://store.steampowered.com/app/1687950/Persona_5_Royal/)) using Proton. I have not looked at those too deeply yet.

### Nvidia dGPU

I have a laptop with an AMD CPU + Nvidia GPU. I was able to get everything setup with regular amdgpu drivers and nvidia-open. Nvidia Prime works fine for OpenGL. However, I have not really been able to control Vulkan applications well. It seems like `vkcube` and `dxvk` always try to run on Nvidia dGPU instead of iGPU, even without `prime-run`.

The AMD iGPU does show up in `vulkaninfo`, and I can probably get things to run on the GPU I want by using a hack and preventing it from seeing my Nvidia dGPU. But it is weird that things do not launch on GPU 0 by default.

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [My Rust std for UEFI open PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
- [Arch Linux](https://archlinux.org/)
- [Arch Wiki](https://wiki.archlinux.org/title/Main_page)
