+++
title = "My Week in Code #29"
description = "Updates regarding PocketBeagle 2 Examples and Rust std for UEFI"
date = "2025-04-13T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "rust", "uefi", "pocketbeagle2", "devicetree"]
+++

Hello everyone. A typical work week. Let's go over everything.

# PocketBeagle 2 Examples

More work went to make the [PocketBeagle 2 Examples](https://github.com/beagleboard/vsx-examples) as educational as possible. The goal of the examples is to teach Linux APIs. Thus the examples try to use the kernel drivers for the different devices as much as possible.

## Make Blinky example lower level

The [Blinky example](https://github.com/beagleboard/vsx-examples/tree/main/PocketBeagle-2/blinky) now uses the sysfs entries in the example file. This should help people understand the sysfs LED subsystem interface.

## Make Seven Segments example lower level

The [Seven Segments example](https://github.com/beagleboard/vsx-examples/tree/main/PocketBeagle-2/seven_segment) now uses the sysfs entries in the example file. This should help people understand the sysfs Seven Segments interface.

## Add more RGB LED Examples

The initial RGB LED example produced different color hues using the [pwm-leds-multicolor](https://github.com/torvalds/linux/blob/master/drivers/leds/rgb/leds-pwm-multicolor.c) driver. However, this was a bit overwhelming when converting to sysfs abstractions. This is because there are quite a few sysfs entries for Multicolor LEDS. So it was decided to add some more straightforward examples for the device.

### Fade Brightness

The fade brightness example uses the `brightness` sysfs entries to perform a PWM fade for a single color in the RGB LED. This helps show how the `brightness` sysfs entry affects the RGB Led.

### Fade Intensity

The fade intensity example uses the `multi_intensity` sysfs entry to perform a PWM fade for the single color in the RGB LED. This helps show how `multi_intensity` can be used to simulate the same behavior as the `brightness` sysfs entry.

### Color Circle

The color circle example demonstrates generating different colors in the color circle. It only generates basic colors, instead of the different color shades.

### Hue

The hue example is the original RGB LED example, and generates various color hues in the RGB LED, giving a nice visual appearance.

## Use GPIO Keys for Buttons

In the original examples, I was using `libgpiod` to read Techlab Cape button inputs. However, I have switched the new examples to use [GPIO keys](https://github.com/torvalds/linux/blob/master/drivers/input/keyboard/gpio_keys.c). The buttons now emit RIGHT and LEFT keycodes.

## Make EEPROM example lower level

The EEPROM parsing is not handled in the example itself rather than hiding in the `beagle_helper`. This allows the example to serve as a good demonstration of how to parse ffi structures in Rust and Python.

## Use PWM Beeper driver for Buzzer

The Buzzer example now uses the [PWM Beeper](https://github.com/torvalds/linux/blob/master/drivers/input/misc/pwm-beeper.c) driver, which allows interacting with the character device, similar to GPIO Keys.

# Send v3 of Export Symbols patch

I have sent [Patch v3 for Export Symbols](https://lore.kernel.org/r/20250411-export-symbols-v3-1-f59368d97063@beagleboard.org). I will also be sending a patch for Beagle Capes soon. The reason for Beagle Capes over MikroBUS is that the capes have a much cleaner history with upstream patches. Additionally, I have a cape that exposes a MikroBUS port ([TechLab Cape](https://www.beagleboard.org/boards/techlab)), which should help ensure that board chaining works well.

# Allow specifying Debian package name in Cargo Packager

I have been using a patched version of Cargo Packager for [bb-imager-rs](https://github.com/beagleboard/bb-imager-rs) since it was missing some features. Since [the patch](https://github.com/crabnebula-dev/cargo-packager/pull/313) to disable Linux desktop entry has been merged, I have created a [PR](https://github.com/crabnebula-dev/cargo-packager/pull/321) to allow manually specifying the name for Debian packages. Once it is merged, I can go back to using the upstream cargo packager.

# Null stdin for UEFI Command

According to the docs in `Command::output`:

> By default, stdout and stderr are captured (and used to provide the
resulting output). Stdin is not inherited from the parent and any attempt by the child process to read from the stdin stream will result in the stream immediately closing.

This was being violated by UEFI which was inheriting stdin by default. While the docs don't explicitly state that the default should be NULL, the behavior seems like reading from NULL.

I have created a [PR](https://github.com/rust-lang/rust/pull/139517) to implement `Stdio::Null` and `Stdio::Inherit` for UEFI.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://github.com/beagleboard/bb-imager-rs)
- [PocketBeagle 2 Examples](https://github.com/beagleboard/vsx-examples)
- [Rust std for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
- [Patch v3 of Export Symbols](https://lore.kernel.org/r/20250411-export-symbols-v3-1-f59368d97063@beagleboard.org)
