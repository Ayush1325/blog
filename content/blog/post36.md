+++
title = "My Week in Code #5"
description = "Updates regarding BeaglePlay and BeagleConnect Freedom Zephyr support, BeagleBoard Rust Imager and Rust std for UEFI"
date = "2024-10-13T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "zephyr", "beagleconnect-freedom", "beagleplay", "rust", "uefi", "beaglev-fire"]
+++


Hello everyone. Another light week here. Let's go over everything.

# Arduino Module for Zephyr

A few weeks ago, I created a [PR](https://github.com/zephyrproject-rtos/zephyr/pull/78667) in Zephyr that adds ADC and PWM support to base devicetree. So we can now clean up the overlay present in the Arduino Module. Here is the [PR](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core/pull/124) for it.

Additionally, while working on this, I encountered that the CI is broken for [Arduino Core API module for Zephyr](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core). So, I created a [PR](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core/pull/125) to fix it.

# BeagleBoard Rust Imager

The [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs) now supports using the original [BeagleBoard Imager](https://www.beagleboard.org/bb-imager) config file. This makes the Rust imager almost feature complete the original (only MacOS support is missing).

# Zephyr

Continuing the general work to improve Zephyr support in Beagle platforms, this week saw some doc improvements along with bug fixes.

## Subg Testing

BeagleConnect Freedom has now been added to build samples against for subg overlay testing. Here is the [PR](https://github.com/zephyrproject-rtos/zephyr/pull/79471).

Adding tests also found a bug in the IEEE802154 shell code, which was fixed in the same PR.

## BeagleV-Fire Doc

I haven't been involved in BeagleV-Fire much, but for anyone unfamiliar, BeagleV-Fire supports Zephyr on its E51 and U54 cores. There were some problems with the documentation, so I ended up fixing it. Here is the [PR](https://github.com/zephyrproject-rtos/zephyr/pull/79511).

## BeaglePlay (CC1352P7) Twister Fix

Continuing the work to improve defaults, I created a [PR](https://github.com/zephyrproject-rtos/zephyr/pull/79405) to only enable IEEE802154 subg in the base devicetree of BeaglePlay CC1352P7. In the same PR, I also found out the [Twister](https://docs.zephyrproject.org/latest/develop/test/twister.html) was broken for BeaglePlay CC1352P7. So, I also ended up fixing it by enabling the UART console in default Kconfig.

# Rust std for UEFI

I plan to speed up Rust std for UEFI development again since it has been a work in progress for far too long. After posting on the Rust Zulip, things have started moving again. As a first step, [getcwd and chdir](https://github.com/rust-lang/rust/pull/129794) PR was approved.

I hope more Rust reviewers interested in UEFI can start getting involved so that I can get everything merged by the end of this year.

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom)
- [My Rust std for UEFI open PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
