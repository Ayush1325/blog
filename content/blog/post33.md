+++
title = "My Week in Code #2"
description = "Updates regarding BeagleBoard Rust Imager, BeaglePlay cc1352p7 Zephyr and MicroPython"
date = "2024-09-22T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "linux", "beagleboard", "zephyr", "rust", "beagleconnect-freedom", "beagleplay"]
+++


Hello everyone. This is the second post in my weekly series. There was some work on bb-imager-rs, mostly my pet project, along with some developments with upstream Zephyr and MicroPython. In short, there were no major developments, but it was still a good week.

# BeagleBoard Rust Imager

A while ago, I started work on rewriting [BeagleBoard Imaging Utility](https://www.beagleboard.org/bb-imager) with the goal of adding support for flashing things other than SD cards. I chose [Rust](https://www.rust-lang.org/) and [Iced](https://iced.rs/) for this work.

Fast forward to the present, and now the [Rust port](https://openbeagle.org/ayush1325/bb-imager-rs) can flash CC1352P7 and MSP430 in [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom). Due to some problems with the original MSP430 USB to UART bridge in BeagleConnect Freedom, it was not working on Windows. I ended up rewriting it to work on Windows. 

However, this still left the problem of providing good instructions for flashing the new firmware. Originally, flashing MSP430 required using [python-msp430-tools](https://github.com/zsquareplusc/python-msp430-tools/tree/master), which is a Python 2 script and can be difficult to use, especially on Windows. Thus, I added the functionality to the imager to allow GUI-based flashing. Additionally, since it is a standalone binary, chasing dependencies is unnecessary.

I also updated to Iced v0.13.1 in the process, which introduced rich text. So anyone interested in porting to new Iced can also use this as a reference.

The Rust-based imager is still missing one critical platform, MacOS, since I do not have a Mac. So, anyone interested is free to take a crack at it.

## Screenshots

Here are some screenshots from the application.

### Home Page
{{ image(src="/images/post33/bbimager_home.webp")}}

### Extra Configuration Page
{{ image(src="/images/post33/bbimager_config.webp")}}

### Flashing Page
{{ image(src="/images/post33/bbimager_flash.webp")}}

# Update BeagleConnect Freedom Zephyr devicetree

Currently, PWM and ADC support is absent in the based devicetree for [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom). This is due to the fact that I added PWM and ADC support quite recently. So, after some deliberation, I think it would be best to enable ADC and PWM in the base devicetree to make for an easier new user experience. It also allows running PWM and ADC-related tests in Twister. Here is the [PR](https://github.com/zephyrproject-rtos/zephyr/pull/78667).

# BeaglePlay CC1352 Zephyr Support Updates

Zephyr support for [BeaglePlay CC1352P7](https://www.beagleboard.org/boards/beagleplay) has been a work in progress for a long time at this point. It was initially planned for v3.7.0, but sadly, it missed that deadline. Forward to the present, and it finally seems to be ready. The name for the target has changed to `beagleplay/cc1352p7` to allow for future support for Zephyr on `M4/A53`.

It already has the required approvals, so I hope it can be merged soon. Here is the [PR](https://github.com/zephyrproject-rtos/zephyr/pull/64718)

# MicroPython Zephyr Mainline Port

Most BeagleBoard boards have some level of MicroPython support. However, since MicroPython usually tracks some stable versions of Zephyr, it can take a long time to add support for the board to upstream MicroPython. This often means by the time things get ready to be merged upstream, the fork with MicroPython is often unmaintained.

After a discussion on MicroPython Discord, I decided to maintain a branch of MicroPython that tracks mainline Zephyr. This can serve as a jumping point for boards still in development or ones merged after the Zephyr version currently supported by MicroPython. Currently, the plan is to keep this branch as a draft PR. I will also keep the MicroPython support for new BeagleBoard boards in this fork until they are ready to be merged into the main branch.

Here is the link to [draft PR](https://github.com/micropython/micropython/pull/15891)

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [BeaglePlay CC1352P7 Zephyr PR](https://github.com/zephyrproject-rtos/zephyr/pull/64718)
- [MicroPython upstream Zephyr PR](https://github.com/micropython/micropython/pull/15891)
