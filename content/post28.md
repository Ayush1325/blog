+++
title = "GSoC23: Summary Post"
description = "A greybus demo with my driver and zephyr application"
date = "2023-08-26T18:15:12+05:30"

[taxonomies]
tags = ["c", "gsoc23", "zephyr", "linux"]
+++
Hello everyone. This is the final blog post I will write before Google Summer of Code 2023 ends (hopefully not the final greybus one). In this post, I will summarize my GSoC23 work to make it more accessible to everyone.

<!-- more -->

# Background
During GSoC23, I worked on replacing [GBridge](https://git.beagleboard.org/beagleconnect/linux/gbridge) by moving SVC and APBridge roles to BeaglePlay CC1352. This allows for simplifying the stack and easier merging to upstream Linux.

The new simplified architecture looks as follows:

{{ image(src="/images/post21/new_diagram.webp")}}

In some rudimentary testing, the new setup can easily handle non-stop remote device usage with a latency of around 40ms for each request/response pair, measuring over 1,000,000 requests without any message drops. Since I only have one Beagleconnect Freedom, I have yet to be able to test how the number of nodes affects the performance.

# Project Links
- [BeaglePlay Linux Driver](https://git.beagleboard.org/gsoc/greybus/beagleplay-greybus-driver)
- [CC1352 Zephyr Application](https://git.beagleboard.org/gsoc/greybus/cc1352-firmware)
- [Proposal](https://elinux.org/BeagleBoard/GSoC/2023_Proposal/AyushSingh)
- [Weekly Progress Report](https://forum.beagleboard.org/t/weekly-progress-report-replace-gbridge/34762/14)

# Project Goals
## Completed
1. Implement Greybus Node Discovery over mDNS.
2. Implement SVC Progocol in the Zephyr Application.
3. Implement APBridge Role in Zephyr Application.
4. Create a Linux driver to serve as AP.
5. Implement HDLC communication between the Linux driver and BeaglePlay CC1352 over UART.
6. Test the complete new BeagleConnect setup with all the parts.

## Partially Completed
1. Get the Linux Driver merged upstream. (Submitted in mailing list but not yet merged)

# Benefits of this Project
- GBridge requires gb-netlink driver, which was never accepted upstream. Eliminating it would allow easier upstreaming of the whole BeagleConnect stack.
- This, in turn, should help with industry adoption by providing strong guarantees as being part of the Linux Kernel
- This will also help simplify the whole BeagleConnect stack, thus making it much easier to get started for Beginners.

# Blog Posts
1. [GSoC 2023 Introduction](@/post21.md)
2. [GSoC23: Writing a Hello World Driver for BeaglePlay](@/post22.md)
3. [GSoC23: Getting started with Zephyr for BeaglePlay CC1352](@/post23.md)
4. [GSoC23: Linux Serial Device Bus](@/post24.md)
5. [GSoC23: Concurrency in ZephyrRTOS](@/post25.md)
6. [GSoC23: Project Status update](@/post26.md)
7. [GSoC23: BeaglePlay Greybus Demo](@/post27.md)

# Videos
## Introduction
{{ yt(id="tCtehnXODW8") }}

## Demo
{{ yt(id="O5coD55JvGU") }}

## Final
{{ yt(id="GVuIB7i5pjk") }}

# Conclusion
I want to thank my mentors and all the members of beagleboard.org Slack and Discord channels. I would also like to thank kernenewbies and greybus mailing list members for helping with my linux driver-related questions. Finally, I would like to thank Google for running this wonderful open-source program.

I will continue working on getting the Linux Driver merged upstream. I also have some more Zephyr and Greybus ideas lined up, which I will be working on in my free time. Feel free to experiment with the new Greybus setup, and contact me if you need any help.

Consider [supporting me](@/pages/supportme.md) if you like my work.
