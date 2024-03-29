+++
title = "GSoC 2023 Introduction"
description = "A introduction post about my GSoC 2023 Project"
date = "2023-05-22T19:07:12+05:30"

[taxonomies]
tags = ["rust", "c", "gsoc23", "linux", "zephyr", "beagleboard"]
+++
Hello everyone. I am Ayush Singh, a third-year student at the Indian Institute of Technology, Dhanbad, India. As a part of Google Summer of Code 2023, I will be working on Replacing GBridge under the [BeagleBoard.org](https://www.beagleboard.org/) organization. In this post, I will describe the project I will be working on.

<!-- more -->

You can follow my GSoC23-related blog posts using this [feed](https://www.programmershideaway.xyz/tags/gsoc23/atom.xml).

# Introduction
## BeagleConnect Technology
[BeagleConnect™](https://docs.beagle.cc/latest/boards/beagleconnect/index.html) is a revolutionary technology that eliminates low-level software development for IoT and IIoT applications, such as building automation, factory automation, etc. 

While numerous IoT and IIoT solutions available today provide massive software libraries for microcontrollers supporting a limited body of sensors, actuators, and indicators, as well as libraries for communicating over various networks, BeagleConnect eliminates the need for these libraries by shifting the burden into the most massive and collaborative software project of all time, the Linux kernel.

## BeagleConnect Freedom
[BeagleConnect™ Freedom](https://docs.beagleboard.org/latest/boards/beagleconnect/freedom/index.html) is an open-hardware wireless hardware platform developed by BeagleBoard.org and built around the TI CC1352P7 microcontroller, which supports both 2.4-GHz and long-range, low-power Sub-1 GHz wireless protocols. 

Rapid prototyping of IoT applications is accelerated by hardware compatibility with over 1,000 mikroBUS add-on sensors, actuators, indicators, and additional connectivity and storage options. It is backed with software support utilizing the Zephyr, allowing developers to tailor the solution to their specific needs. 

BeagleConnect Freedom further includes MSP430F5503 for USB-to-UART functionality, temperature and humidity sensor, light sensor, SPI flash, battery charger, buzzer, LEDs, and JTAG connections to make it a comprehensive solution for IoT development and prototyping. 

## Beagle Play
[BeaglePlay](https://docs.beagleboard.org/latest/boards/beagleplay/index.html) is an open-source single-board computer designed to simplify the process of adding sensors, actuators, indicators, human interfaces, and connectivity to a reliable embedded system. 

It features a powerful 64-bit, quad-core processor and innovative connectivity options, including WiFi, Gigabit Ethernet, sub-GHz wireless, and single-pair Ethernet with power-over-data-line. With compatibility with 1,000s of off-the-shelf add-ons and a customized Debian Linux image, BeaglePlay makes expansion and customization easy. 

It also includes ribbon-cable connections for cameras and touch-screen displays, and a socket for a battery-backed real-time clock, making it ideal for human-machine interface designs.

# Project Goals
- Introduce a Platform Driver that facilitates communication between AM6254 and CC1352 in BeaglePlay over UART.
- Move SVC and APBridge roles into CC1352.
- Eliminate [GBridge](https://git.beagleboard.org/beagleconnect/linux/gbridge) and other components that are no longer necessary.
- Add a tutorial to the documentation to use the new setup.

# Current Architecture
Here is the diagram of the current architecture:
{{ image(src="/images/post21/old_diagram.webp")}}

## Limitations
1. The current architecture revolves around [GBridge](https://git.beagleboard.org/beagleconnect/linux/gbridge). While this makes it quite flexible, it adds a level of indirection. Also, it has been challenging to get [gb-netlink](https://git.beagleboard.org/beagleconnect/linux/gbridge) merged upstream, hindering the adoption of BeagleConnect.
2. Makes the setup confusing for beginners.


# New Architecture
Here is a simple diagram of the new architecture:
{{ image(src="/images/post21/new_diagram.webp")}}

Due to the availability of Beagle Play (which contains both AM6254 and CC1352 in one board), the architecture can be simplified to make BeagleConnect truly plug and play.

Now I will go into more detail about the different components of the new architecture.

## SVC Role
The cc1352 firmware will need to detect the hotplug and removal of modules. In the current architecture, each controller runs an event_loop in a `pthread` to perform its functionality. I am thinking of using a [mDNS](https://docs.zephyrproject.org/3.2.0/connectivity/networking/api/dns_resolve.html) based approach:

1. Run a loop in a separate thread that polls for `_greybus._tcp`.
2. Maintain a table of nodes.
3. Compare the polled devices against the table for new/removed nodes.
4. Handle new devices and free removed device resources.

Initially, I am planning on implementing a basic version using [threads](https://docs.zephyrproject.org/3.2.0/kernel/services/threads/index.html). However, I will write a more efficient implementation using the [Polling API](https://docs.zephyrproject.org/3.2.0/kernel/services/polling.html) if I have time.

## AP Bridge Role
The cc1352 firmware will now also be responsible for the APBridge role, which GBridge currently handles. This role involves a variety of tasks:

- Reading/Writing Greybus messages over 6lowpan and generating messages to send to the host.
- Installing/Uninstalling greybus interfaces on insertion/removal after receiving an event from SVC.
- Get Device manifest on insertion.
- Sending events to register drivers, etc.

The initial implementation will probably use simple threads. However, the final goal is to write the implementation using [MessageQueues](https://docs.zephyrproject.org/latest/kernel/services/data_passing/message_queues.html#message-queues) + [Events](https://docs.zephyrproject.org/latest/kernel/services/synchronization/events.html) which Zephyr supports.

## Greybus Serdev Driver

This new driver will be responsible for communication between Greybus and cc1352. This will be a Serial Device Bus Driver (serdev) driver with the following structure:

```c
struct serdev_device_driver {
	struct device_driver driver;
	int	(*probe)(struct serdev_device *);
	void	(*remove)(struct serdev_device *);
};
```

The actual types that are used in communication can be found at [`include/linux/greybus/greybus_protocols.h`](https://github.com/torvalds/linux/blob/master/include/linux/greybus/greybus_protocols.h) in upstream Linux kernel.

The driver will run on BeaglePlay's AM62 processor, although it should be possible on any Linux host.

The two main functions of this driver are as follows:

### probe

This step will involve the following:

- Initialize stuff: This includes Work Queue, spin locks, and allocate read/write buffers.
- Open the serial device. Set parameters such as the baud rate. Mostly copy the configuration from bcfserial.
- Create gb_host_device.

Since we only want to use a single static UART (between AM62 and CC1352), we can optimize the driver probe.

### remove

- Flush the contents of the work queue.
- Close the serdev device

# Benefits of this Project

- [GBridge](https://git.beagleboard.org/beagleconnect/linux/gbridge) requires [gb-netlink](https://git.beagleboard.org/beagleconnect/linux/greybus) driver, which was never accepted upstream. Eliminating it would allow easier upstreaming of the whole BeagleConnect stack.
- This, in turn, should help with industry adoption by providing strong guarantees as being part of the Linux Kernel
- This will also help simplify the whole BeagleConnect stack, thus making it much easier to get started for Beginners.

# Conclusion
I am excited to work on this project since I have always wanted to do embedded development. I will probably write a few posts detailing the setup I will be using.

Consider [supporting me](@/pages/about.md) if you like my work.

# Project Links
- Proposal: [https://elinux.org/BeagleBoard/GSoC/2023_Proposal/AyushSingh](https://elinux.org/BeagleBoard/GSoC/2023_Proposal/AyushSingh)
- Project Tracker: [https://forum.beagleboard.org/t/weekly-progress-report-replace-gbridge/34762](https://forum.beagleboard.org/t/weekly-progress-report-replace-gbridge/34762)
