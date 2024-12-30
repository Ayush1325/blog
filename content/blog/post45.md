+++
title = "My Week in Code #14"
description = "Updates regarding Pocketbeagle2, BeagleConnect Freedom and R-EFI"
date = "2024-12-29T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "zephyr", "rust", "pocketbeagle2", "beagleconnect-freedom"]
+++

Hello everyone. I have been traveling for the last few weeks due to some family functions. Due to this, I could not post [My Week in Code](/tags/weekly-update/). Additionally, I was primarily working on [Pocketbeagle2](https://www.beagleboard.org/boards/pocketbeagle-2) stuff, which wasn't public yet, so I also lacked content. Now that Pocketbeagle2 has been announced, I can start talking about it. So, let's get to it.

# Pocketbeagle2 MSPM0 Driver

In [Pocketbeagle2](https://www.beagleboard.org/boards/pocketbeagle-2), instead of using dedicated EEPROM and ADC, we are using [MSPM0L1105](https://www.ti.com/product/MSPM0L1105) with [specilized firmware](https://openbeagle.org/pocketbeagle/mspm0-adc-eeprom) that acts as EEPROM and ADC.

While it is possible to use a [userspace flasher](https://openbeagle.org/pocketbeagle/msp-m0-python-flasher), it is much easier to perform firmware updates with an upstream driver. Additionally, it also allows doing some custom pinmux and overlay stuff, which might be required due to a problem with the nRST line.

While the MSPM0L1105 is intended to be used as EEPROM and ADC by default, it can be used as a dedicated co-processor for IO and communicate over I2C or SPI. I will also try to add Zephyr support in the future.

The driver is currently in [my kernel tree](https://github.com/Ayush1325/linux/commits/b4/pb2-mspm0/).

# Update Ti simplelink SDK in hal_ti

A while ago, I found an issue with BeagleConnect Zephyr support, where everything freezes if both PWM and SUBg are enabled at the same time. Thanks to help from [Arthur](https://e2e.ti.com/members/7092658), I was able to confirm that this does not happen in Ti-RTOS and is thus a problem with Zephyr implementation. However, I was not able to find the root cause, even with JTAG debugging. The complete discussion can be found in [e2e](https://e2e.ti.com/support/wireless-connectivity/sub-1-ghz-group/sub-1-ghz/f/sub-1-ghz-forum/1430241/cc1352p7-beagleconnect-freedom-freezes-if-both-pwm-and-ieee802154-subg-is-used-together/5584015#5584015).

Fast forward to last week, and I thought I should try tackling it again before starting work on Arduino support using LLEXT. One of the possibilities I  came up with was that it might be a bug in the version of simplelink SDK that was being used in [hal_ti](https://github.com/zephyrproject-rtos/hal_ti/pull/56). The SDK being used currently is 4.40.04.04, which is quite old at this point. Additionally, I wanted to use the latest SDK before attempting to tackle Bluetooth again.

While working on it, I realized that it is currently not possible to automate SDK updates since we need to make some modifications manually. In the end, it turned out that the SDK version was not the cause of the freeze, but it helped me figure out the real cause.

Hopefully, I can figure out a more sustainable way moving forward, but I just went with the manual route for now.

It also seems like GPIO APIs have completely changed, and I had to fix some stuff in the Zephyr GPIO driver. Here are the open PRs: [hal_ti](https://github.com/zephyrproject-rtos/hal_ti/pull/56) and [zephyr](https://github.com/zephyrproject-rtos/zephyr/pull/83402).

# Fix BeagleConnect Freedom Freeze Bug

As discussed above, I was finally able to pinpoint the cause of the freeze. It turns out that the Zephyr power management was at fault. Both the PWM driver and RF core driverlib can put and release constraints on the suspend state of cc1352p7. However, at least the PWM code would release the constraint without checking if PWM placed the constraint in the first place.

I have created an initial [PR](https://github.com/zephyrproject-rtos/zephyr/pull/83409) to fix the problem by ensuring that PWM keeps track of the suspend constraint. This ensures that PWM does not cause the processor to go to suspend even if it has been disabled by subg code.

# Rust UEFI varargs function support

For the longest time, Rust ffi had the annoying limitation of not supporting varargs in ABIs other than C. However, recently `extended_varargs_abi_support` was [stablised](https://github.com/rust-lang/rust/pull/116161). So finally, it is possible to have correct function signatures for `BootServices->{Install/Uninstall}MultipleProtocolInterfaces`. Here is the [PR](https://github.com/r-efi/r-efi/pull/87) in r-efi.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [Pocketbeagle2](https://openbeagle.org/pocketbeagle/pocketbeagle-2)
- [Pocketbeagle2 mspm0 driver tree](https://github.com/Ayush1325/linux/commits/b4/pb2-mspm0/)
