+++
title = "My Week in Code #7"
description = "Updates regarding BeagleConnect Freedom, MicroPython, Zephyr, BeagleBoard Rust Imager and Rust std for UEFI"
date = "2024-10-27T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "zephyr", "beagleconnect-freedom", "beaglebone-ai64", "beagley-ai", "beagleplay", "linux", "rust", "uefi"]
+++

Hello everyone. This week mostly involved a lot of chasing stuff around (sometimes in vain), so while there was not much headline work, this post might end up a bit longer than usual. Let's get started without delay.

# BeagleConnect Freedom Adventures

I started the week by trying to add IEEE802154 subg socket support in MicroPython for [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom). However, I quickly learned that it would not be simple. For some reason, BeagleConnect Freedom would completely lock up after starting. 

Initially, I thought it might be a MicroPython issue, so I tried tracking down where the freeze happened. This, however, led to a dead end since, for some reason, the program would not be able to read from the UART console. While doing this, I also remembered a similar issue I encountered while working on [BeagleConnect Freedom rc car demo](https://openbeagle.org/ayush1325/rc-car). At the time, I fixed it by just removing unused config items like ADC and PWM from config, but forgot about it after the [OOSC conference](https://events.canonical.com/event/89/).

After some experimenting with PWM, ADC, and IEEE802154 subg radio, I figured out that the problem is reproducible in other Zephyr samples like [echo_cliet](https://docs.zephyrproject.org/latest/samples/net/sockets/echo_client/README.html#sockets-echo-client), etc. For some reason, if both PWM pins (MB1 PWM and MB2 PWM) are enabled alongside the subg radio, everything freezes. If one or both of the PWM are disabled, everything works fine. This seems to be an issue with timers but it needs further investigation.

I have created a [Zephyr issue](https://github.com/zephyrproject-rtos/zephyr/issues/80252) and a [Ti E2E question](https://e2e.ti.com/support/wireless-connectivity/sub-1-ghz-group/sub-1-ghz/f/sub-1-ghz-forum/1430241/cc1352p7-beagleconnect-freedom-freezes-if-both-pwm-and-ieee802154-subg-is-used-together) for the same.

# Code Composer Studio Theia Adventures

With the MicroPython issue and the [bricked BeagleConnect Freedom](@/blog/post35.md#support-for-more-firmware-file-types), I thought it is a good time to setup and learn Ti's [Code Composer Studio](https://www.ti.com/tool/CCSTUDIO).

I use [Fedora Sway Atomic](https://fedoraproject.org/atomic-desktops/sway/) as my daily driver, and thus mostly rely on flatpaks or podman containers. However, running Code Composer Studio inside a podman container (created using [toolbox](https://docs.fedoraproject.org/en-US/fedora-silverblue/toolbox/)) was not a great experience for me. It would randomly stutter (maybe a hardware acceleration problem?) and freeze. Additionally, while udev can make it almost painless to handle device permissions, it can occasionally cause hiccups with flashing. In fact, one of the primary reasons I switched to [neovim](https://neovim.io/) was that my [emacs](https://www.gnu.org/software/emacs/) GUI kept having weird performance problems inside the container.

So, I finally went ahead and installed CCS Theia on my base system. The install procedure is a bit weird since there is no `rpm` or `deb` package. Instead, there is an installer which installs everything in `$HOME/ti` folder. It also creates an uninstall, which seems to work. All in all, while I prefer a flatpack or app image, it wasn't too bad.

I hit a snag quite early on when I was unable to flash the cc1352p1 on my [launchpad](https://www.ti.com/tool/LAUNCHXL-CC1352P). I tried various things and opened a [Ti E2E question](https://e2e.ti.com/support/processors-group/processors/f/processors-forum/1428729/launchxl-cc1352p-cannot-flash-using-ccs-theia) for the same. However, the solution turned out to be quite weird. I was not saving my workspace since, well, nothing was working anyway, and CCS Theia would open the unsaved workspace. But everything magically worked once I saved my workspace because I was tired of the dialog on exit. Not really sure why.

Once I could flash the launchpad, I tried using the [xds110](https://software-dl.ti.com/ccs/esd/documents/xdsdebugprobes/emu_xds110.html) in launchpad with my BeagleConnect Freedom. I was able to flash a simple blinky on it and even set up breakpoints.

Now, I need to figure out how to use [openocd](https://openocd.org/) and add instructions in Beagle docs and Zephyr docs for the same.

# KUnit Adventures

I have been working on kernel patches that require writing some unit tests. So I was trying to get [KUnit](https://www.kernel.org/doc/html/latest/dev-tools/kunit/index.html) to work. However, `kunit run` kept on failing for some reason, even with the default config. The output was not very clear either. However, after following some debugging instructions, I found out that I could not execute the user mode kernel from inside the podman container. I have created an [issue](https://github.com/containers/toolbox/issues/1574) in Fedora toolbox regarding the same.

# MicroPython

I have added MicroPython support for [BeaglePlay cc1352p7](https://www.beagleboard.org/boards/beagleplay) in my [draft PR](https://github.com/micropython/micropython/pull/15891). It supports IEEE802154 subg sockets and also helped me ensure that MicroPython networking should work fine on BeagleConnect Freedom as well once the timer issue is resolved.

Since BeaglePlay cc1352p7 Zephyr support was merged after the 3.7.0 release, the MicroPython support will continue to live in the draft PR until MicroPython supports a newer Zephyr version.

# Zephyr

Zephyr support for [BeagleBoard](https://www.beagleboard.org/) boards continues to improve. We will continue to work to make Beagle one of the best-supported platforms for Zephyr development.

## BeagleBone AI-64

Thanks to the work by [Andrew Davis](https://github.com/glneo), Zephyr support for R5 cores in BeagleBone AI64 was merged this week. Here is the [Zephyr page for BeagleBone AI 54](https://docs.zephyrproject.org/latest/boards/beagle/beaglebone_ai64/doc/index.html). This adds one more board to the growing list of BeagleBoard boards that support Zephyr.

## BeagleY-AI

A [PR](https://github.com/zephyrproject-rtos/zephyr/pull/80344) for Zephyr support was opened by [Andrew Davis](https://github.com/glneo) after BBAI-64 support was merged. Anyone interested should feel free to try it out. Hopefully, it can get merged upstream soon.

# BeagleBoard Imager Rust Updates

While working on BeagleY-AI, I found a bug in the sha256 handling of the Rust-based imager while translating the old bb-imager config. So, I have created [release 0.0.2](https://openbeagle.org/ayush1325/bb-imager-rs/-/releases/v0.0.2) for the imager. I probably should implement changelogs before the next release.

# Rust Path Prefix

While working on File I/O support for UEFI, I was reminded again of the limitations of [std::path::Prefix](https://doc.rust-lang.org/std/path/enum.Prefix.html). To fix this, an issue has been opened in [Github](https://github.com/rust-lang/rust/issues/52331), and I have posted my [proposal](https://github.com/rust-lang/rust/issues/52331#issuecomment-2439721420). Feel free to add suggestions to it.

I will start implementing the proposal once I deem the File I/O support good enough to mark the [draft PR](https://github.com/rust-lang/rust/pull/129700) as ready.

# Migrating to dev domain

I have migrated my blog from [https://programmershideaway.xyz/](https://programmershideaway.xyz/) to [https://programmershideaway.dev/](https://programmershideaway.dev/). The reason I started out with xyz was because it was extremely cheap at the time. However, now that I am writing regularly, I have decided to switch to `.dev`, since that feels better than `.xyz` for a programming blog. I have also setup redirects from the old domain, so all the old links will continue to work for now. I will renew the old domain for at least one more year. Hopefully, by 2026, all the old links will be updated to the new one.

I used [Cloudflare Registrar](https://www.cloudflare.com/en-in/products/registrar/) for the domain (since my old one is also here), but payments were really difficult to do from an Indian debit card. I hope Cloudflare can fix this by the time I have to renew this domain.

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [Code Composer Studio](https://www.ti.com/tool/CCSTUDIO)
- [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom)
- [BeaglePlay](https://www.beagleboard.org/boards/beagleplay)
- [BeagleBone AI-64](https://www.beagleboard.org/boards/beaglebone-ai-64)
- [BeagleY-AI](https://www.beagleboard.org/boards/beagley-ai)
