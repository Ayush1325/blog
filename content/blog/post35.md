+++
title = "My Week in Code #4"
description = "Updates regarding BeaglePlay and BeagleConnect Freedom Zephyr support, MicroPython and BeagleBoard Rust Imager"
date = "2024-10-06T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "zephyr", "rust", "beagleconnect-freedom", "beagleplay"]
+++

Hello everyone. Another light week here. Let's go over everything.

# BeagleConnect Freedom

## Zephyr

While [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) has had good Zephyr support for a while, the base devicetree had some missing features that would require overlays. This would sometimes cause confusion among beginners. So, I created a [PR](https://github.com/zephyrproject-rtos/zephyr/pull/78667) to provide better defaults.

I also created another [PR](https://github.com/zephyrproject-rtos/zephyr/pull/79407) regarding some other cleanup.

## MicroPython

MicroPython support for Zephyr v3.7.0 was finally [merged](https://github.com/micropython/micropython/pull/9335) last week. I would like to thank [Maureen Helm](https://github.com/MaureenHelm) for all the amazing work.

I have created a [PR](https://github.com/micropython/micropython/pull/15959) to add BeagleConnect Freedom support to upstream MicroPython. It only supports SPI, I2C, and FLASH for now, but once the base support is accepted, I will expand to include support for everything possible with v3.7.0.

# BeaglePlay Zephyr

[BeaglePlay cc1352p7](https://docs.zephyrproject.org/latest/boards/beagle/beagleplay/doc/beagleplay_cc1352p7.html) support was finally merged last week. While it was initially scheduled for Zephyr v3.7.0, it ended up being delayed. However, I hope that having upstream Zephyr support helps people create some amazing projects.

Similar to BeagleConnect Freedom, I also cleaned up the BealgePlay devicetree. Here is the [PR](https://github.com/zephyrproject-rtos/zephyr/pull/79405)

# BeagleBoard Imager Rust Utility

Continuing from [last week](@/blog/post34.md), there was a lot more development on the Rust-based imager utility.

## Support for more firmware file types

BeagleBoard Rust Imager now supports flashing [Ti-TXT](https://downloads.ti.com/docs/esd/SPRUI03/ti-txt-hex-format-ti-txt-option-stdz0795656.html), [Intel Hex](https://www.intel.com/content/www/us/en/programmable/quartushelp/17.0/reference/glossary/def_hexfile.htm) to [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom). This should help with using BeagleConnect Freedom using CCStudio.

I also ended up flashing a firmware that disabled BSL, so I need to figure out how to flash a firmware that enables BSL again.

## Better Cross compile

I want the imager utility to work on aarch64, specifically on BeaglePlay, BeagleY-AI, etc. However, the dependence on OpenSSL and libudev made cross-compiling a bit difficult. 

I was able to simplify local development using [cross](https://github.com/cross-rs/cross). However, using cross on Gitlab CI was much more difficult than I initially expected. Additionally, BeagleBoard Gitlab only had one Docker-in-Docker runner, so I opted for a more manual approach. My [CI config](https://openbeagle.org/ayush1325/bb-imager-rs/-/blob/main/.gitlab-ci.yml?ref_type=heads) should serve as a good starting point for anyone in a similar situation.

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom)
