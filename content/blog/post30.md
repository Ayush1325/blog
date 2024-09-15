+++
title = "What is BeagleConnect Technology"
description = "A short overview of BeagleConnect Technology"
date = "2024-01-19T00:30:12+05:30"

[taxonomies]
tags = ["beagleconnect", "zephyr", "beagleboard"]
+++

Hello everyone. I have been a bit busy these past few months working to improve BeagleConnect™ Technology. While this work initially started as a Google Summer of Code 2023 project, I am now working part-time on this. While not everything is completed, I felt this was a good point to go through an overview of it before I start creating demos (which are actually reproducable without having to approach me). So here it goes.

<!-- more -->

# Brief Pitch

BeagleConnect™ is a revolutionary technology virtually eliminating low-level  software development for [IoT](https://en.wikipedia.org/wiki/Internet_of_things) and [IIoT](https://en.wikipedia.org/wiki/Industrial_internet_of_things) applications, such as building automation, factory automation, home automation, and scientific data acquisition. 

While numerous IoT and IIoT solutions available today provide massive software libraries for microcontrollers supporting a limited body of [sensors](https://en.wikipedia.org/wiki/Sensor), [actuators](https://en.wikipedia.org/wiki/Actuator) and [indicators](https://en.wikipedia.org/wiki/Indicator_(distance_amplifying_instrument)) as well as libraries for communicating over various networks, BeagleConnect simply eliminates the need for these libraries by shifting the burden into the  most massive and collaborative software project of all time, the [Linux kernel](https://en.wikipedia.org/wiki/Linux_kernel).

These are the tools used to automate things in [scientific data collection](https://en.wikipedia.org/wiki/Data_collection_system), [data science](https://en.wikipedia.org/wiki/Data_science), [mechatronics](https://en.wikipedia.org/wiki/Mechatronics), and [IoT](https://en.wikipedia.org/wiki/Internet_of_things).

BeagleConnect™ technology solves:

- The need to write software to add a large set of diverse devices to your system,
- The need to maintain the software with security updates,
- The need to rapidly prototype using off-the-shelf software and hardware without wiring,
- The need to connect to devices using long-range, low-power wireless, and
- The need to produce high-volume custom hardware cost-optimized for your requirements.

Now to get into more nitty-gritty details, we will first need some information about Greybus, which powers BeagleConnect™ Technolgy.

# Greybus

I will be taking information from the [Greybus LWN article](https://lwn.net/Articles/715955/). So feel free to check it out.

Greybus was initially designed for Google's Project Ara smartphone (which is discontinued now), but the first (and only) product released with it is Motorola's Moto Mods. It was initially merged for potential use by kernel components that need to communicate in a platform-independent way.

The [Greybus specification](https://github.com/projectara/greybus-spec) provides device discovery and description at runtime, network routing and housekeeping, and class and bridged PHY protocols, which devices use to talk to each other and to the processors. The following figure shows how various parts of the kernel interact with the Greybus subsystem.

{{ image(src="/images/post30/greybus.webp")}}

There are three main entities in the Greybus network:

1. **AP:** It refers to the host CPUs, i.e., CPUs running Linux in most cases. It is responsible for administrating the Greybus network via the SVC.
2. **SVC:** The SVC represents an entity within the Greybus network that configures and controls the Greybus (UniPro) network, mostly based on the instructions from the AP. All module insertion and removal events are first reported to the SVC, which in turn informs the AP about them using the SVC protocol.
3. **Module:** A module is the physical hardware entity that can be connected or disconnected statically (before powering the system on) or dynamically (while the system is running) from the Greybus network. Once the modules are connected to the Greybus network, the AP and the SVC enumerate the modules and fetch per-interface manifests to learn about their capabilities.

While Greybus is a great protocol, the implementation is tightly coupled with the UniPro transport. This makes it challenging to use Greybus in other modes of transport.

# BeagleConnect™ Technology

BeagleConnect™ Technology aims to use Greybus outside of the traditional Greybus network. This includes using transports other than UniPro (such as 6lowpan), using embedded devices running [ZephyrRTOS](https://zephyrproject.org/) as modules, emulating SVC in co-processor, etc. This makes BeagleConnect™ much more flexible than what traditional greybus seems to support. Here is a diagram of the general BeagleConnect™ setup:

{{ image(src="/images/post30/SoftwareProp.webp")}}

The SVC is either emulated in userspace software in the SOC (gbridge) or in a co-processor (e.g., in [BeaglePlay](https://www.beagleboard.org/boards/beagleplay)). The arbitrary transport can be anything from 6lowpan (for long range) to ethernet or optical cables (for max speed). Finally, greybus nodes such as BeagleConnect Freedom running Greybus Zephyr firmware allow the use of [mikroBUS](https://www.mikroe.com/mikrobus) which opens a host of Plug and Play possibilities for peripherals.

# Why should you use BeagleConnect™?

1. **Open-source:** The [Greybus Spec](https://github.com/projectara/greybus-spec) is open-source and a part of the Linux kernel. This makes it easy to use and personalize for your use case. Being part of the Linux Kernel also provides it a level of reliability that most similar solutions lack.

2. **Network agnostic:** BeagleConnect™ allows Greybus to be network agnostic. This means it can be used over networks like 6lowpan, which has incredible wireless range, or over optical networks for high-throughput, low-latency use cases.

3. **Rapid Prototyping:** Any device (e.g., [clickboard](https://www.mikroe.com/click-boards)) connected to the greybus node can be accessed from the Linux host. In this setup, only the Linux host needs to have device drivers. The greybus node (running ZephyrRTOS, nuttx, etc) does not need the device drivers. This allows being able to prototype devices by just creating a Linux driver instead of having to write drivers for each individual embedded OS.

4. **Star topology IoT and IIoT networks:** In most star topology IoT networks, the only job of peripheral nodes is to collect data from connected devices, convert it to a common format (such as JSON), and then send it to the central node for further processing. Using BeagleConnect™ Technology allows us to use a low-powered device for nodes (since the data does not need to be processed much). Additionally, nodes can be swapped much more easily since anything can be a Greybus node as long as it implements the protocol.

# Conclusion

I have been working these past few months to move everything to the new architecture with [gb-beagleplay](https://elixir.bootlin.com/linux/latest/source/drivers/greybus/gb-beagleplay.c) driver (part of mainline 6.7). Most of the Zephyr patches that were needed are already upstreamed or have open PRs. So things should be more or less for tinkerers soon. I will also post updates on this blog, so feel free to follow the [atom feed](http://127.0.0.1:1111/tags/beagleconnect/atom.xml). You can also check out my [GSoC23 Summary Post](@/blog/post28.md) which provides an overview of my GSoC work along with some demos.

We are looking for more people to join us in improving and/or exploring BeagleConnect™ technology. Feel free to reach out to us at [Discord](https://discordapp.com/channels/1108795636956024986/1189277127590289469) or [BeagleBoard Forum](https://forum.beagleboard.org/).

Consider [supporting me](@/pages/about.md) if you like my work.
