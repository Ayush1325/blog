+++
title = "My Week in Code #1"
description = "A look at some stuff from the past week"
date = "2024-09-15T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "linux", "beagleboard", "microblocks", "beagleplay", "beagleconnect-freedom", "beagley-ai"]
+++

Hello everyone. I am starting a new series of weekly posts to go over stuff I worked on/found interesting over the week. I will try to post one every Sunday, but I might skip uneventful weeks.

Since I work as an embedded systems engineer at BeagleBoard.org, many updates will be about BeagleBoard stuff. Still, I might throw some Rust std and other things in occasionally. Let's see how this goes.

# MikroBUS Patches

I finally sent v1 of [MikroBUS patches](https://lore.kernel.org/all/20240911-mikrobus-dt-v1-0-3ded4dc879e7@beagleboard.org/) using the approach described by [Andrew Davis](https://lore.kernel.org/linux-arm-kernel/20240702164403.29067-1-afd@ti.com/). I got the approach to work for GPIO, UART, and I2C but not for SPI.

I wrote the driver in Rust to see how it would go, and it was a pretty good experience. Danilo Krummrich will post patches for platform device and driver abstractions for Rust, so any future patches will probably not need to carry those either.

There is also a talk in LPC this week: [Runtime hotplug on non-discoverable busses with device tree overlays](https://lpc.events/event/18/contributions/1696/), so I will be looking forward to it.

# Beagle-Y-AI Hacking Sessions

We at BeagleBoard have started weekly hacking sessions for Beagle-Y-AI. Currently, we are working on adding Zephyr support for the R5 core, so feel free to join in. Here is the [event invite](https://discord.gg/AGU2VXE3?event=1283128283000606751).

# GPIO Nexus Node dtschema

In the MikroBUS patches, I was asked to put the GPIO nexus node dtschema in the upstream dtschema. So just a basic [PR](https://github.com/devicetree-org/dt-schema/pull/143) to add dtschema for GPIO nexus nodes.

# PWM in BeaglePlay

While working on MikroBUS patches, I realized that PWM support was not enabled in the upstream device tree for BeaglePlay. So, I decided to send a [patch](https://lore.kernel.org/r/20240913-beagleplay-pwm-v1-1-d38ee5b36d8c@beagleboard.org) to enable it.

Once the patch is added to the kernel tree, it will be possible to use the PWM pin in the MikroBUS connector as described in [Linux kernel docs](https://docs.kernel.org/driver-api/pwm.html).

# Fix MicroBlocks for BeagleConnect Freedom

Around two weeks ago, the CI [MicroBlocks for BeagleConnect Freedom](https://openbeagle.org/beagleboard/microblocks) broke. After checking the CI logs, it seems [MicroBlocks](https://microblocks.fun/) started using [digitalPinToInterrupt](https://www.arduino.cc/reference/en/language/functions/external-interrupts/digitalpintointerrupt/), which was missing in [Arduino Module for Zephyr](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core). I created a [PR](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core/pull/121) for it, and now the CI is fixed again.

# Ending Thoughts

That is all for the week. Hopefully, I can stick to the weekly schedule.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [MikroBUS Patches](https://lore.kernel.org/all/20240911-mikrobus-dt-v1-0-3ded4dc879e7@beagleboard.org/)
- [Beagle-Y-AI Hacking Sessions](https://discord.gg/AGU2VXE3?event=1283128283000606751)
- [GPIO Nexus Node dtschema PR](https://github.com/devicetree-org/dt-schema/pull/143)
- [PWM in BeaglePlay Patch](https://lore.kernel.org/r/20240913-beagleplay-pwm-v1-1-d38ee5b36d8c@beagleboard.org)
- [MicroBlocks for BeagleConnect Freedom](https://openbeagle.org/beagleboard/microblocks)
