+++
title = "MCUboot with BeagleConnect Freedom"
description = "A demo of using MCUboot based OTA with BeagleConnect"
date = "2025-08-18T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "beagleconnect-freedom", "zephyr"]
+++

Hello everyone. It has been a while since my last post. Today, I will go over using [MCUboot](https://docs.mcuboot.com/) based serial OTA on [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom). Everything required has already been merged to upstream Zephyr now, so no special considerations should be required.

# Introduction

[BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) contains a simple bootloader in ROM, which boots the Zephyr application from the Flash. However, this minimal bootloader does not support advanced functionality such as OTA updates, rollbacks, etc. 

Zephyr supports [MCUboot](https://docs.mcuboot.com/), which is a secure bootloader for 32-bits microcontrollers. It helps add support for advanced production usecases such as secure boot, OTA, rollbacks, etc.

In this post, I will only be going over OTA over serial. However, it is possible to support OTA over IEEE802154g and other transport protocols.

# MCUboot storage use

When using [MCUboot](https://docs.mcuboot.com/) with [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom), 56KiB is reserved for MCUboot. This leaves 640KiB of main flash for use by the Zephyr application. 

The on-board SPI flash is used to store the secondary image. So outof 2MiB, 1280KiB is available for Zephyr application use.

# Demo

For the demo application, I will be using the [smp_svr](https://docs.zephyrproject.org/latest/samples/subsys/mgmt/mcumgr/smp_svr/README.html) example provided by Zephyr.

## Setup Zephyr

Follow the [Getting Started Guide](https://docs.zephyrproject.org/latest/develop/getting_started/index.html) to set up Zephyr on your machine. Currently, mainline Zephyr is required for out-of-the-box MCUboot support on BeagleConnect Freedom.

## Setup AuTerm

Download and set up AuTerm as described in its [repository](https://github.com/thedjnK/AuTerm).

## Build Zephyr Application

1. Build smp_svr + MCUboot using [sysbuild](https://docs.zephyrproject.org/latest/build/sysbuild/index.html).

```sh
west build -p -b beagleconnect_freedom --sysbuild samples/subsys/mgmt/mcumgr/smp_svr -- -DEXTRA_CONF_FILE="overlay-serial.conf"
```

2. Combine the MCUboot and application binary. The application offset should be 56KiB.

```sh
cp build/mcuboot/zephyr/zephyr.bin zephyr.bin
dd conv=notrunc bs=1024 seek=56 if=build/smp_svr/zephyr/zephyr.signed.bin of=zephyr.bin
```

3. Flash the application.

```sh
cc1352_flasher --bcf zephyr.bin
```

4. Check serial output.

```sh
❯ tio /dev/ttyACM0
[11:52:04.720] tio 3.9
[11:52:04.720] Press ctrl-t q to quit
[11:52:04.720] Connected to /dev/ttyACM0
** Booting MCUboot v2.2.0-104-gbf5321bb6906 ***
*** Using Zephyr OS build v4.2.0-1763-gffb28eed1da9 ***
*** Booting Zephyr OS build v4.2.0-1763-gffb28eed1da9 ***
[00:00:00.004,699] <inf> smp_sample: build time: Aug 18 2025 11:51:48
```

## Perform Serial OTA

1. Launch AuTerm.
{{ image(src="/images/post68/1_Serial_OTA.webp") }}

2. Select BeagleConnect Freedom Port.
{{ image(src="/images/post68/2_Serial_OTA.webp") }}

3. Verify the port.
{{ image(src="/images/post68/3_Serial_OTA.webp") }}

4. Open the Port. This will drop you into the Terminal tab. Take note of the build time.
{{ image(src="/images/post68/4_Serial_OTA.webp") }}

5. Press the reset button on BeagleConnect Freedom and verify the serial output.
{{ image(src="/images/post68/5_Serial_OTA.webp") }}

6. Rebuild the firmware as described [above](#build-zephyr-application)

7. Navigate to the MCUmgr tab.
{{ image(src="/images/post68/6_Serial_OTA.webp") }}

8. Select the newly built signed firmware.
{{ image(src="/images/post68/7_Serial_OTA.webp") }}

9. Switch action from Test to Confirm.
{{ image(src="/images/post68/8_Serial_OTA.webp") }}

10. Press the Go button to start firmware upload.
{{ image(src="/images/post68/9_Serial_OTA.webp") }}

11. Wait for the firmware upload to finish.
{{ image(src="/images/post68/10_Serial_OTA.webp") }}

12. Navigate to the Terminal tab to verify that the build time has changed.
{{ image(src="/images/post68/11_Serial_OTA.webp") }}

# Ending Thoughts

That is all for this post. Hopefully, this serves as an example for anyone trying to have OTA working with BeagleConnect Freedom.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom)
- [MCUboot](https://docs.mcuboot.com/)
- [Zephyr smp_svr sample](https://docs.zephyrproject.org/latest/samples/subsys/mgmt/mcumgr/smp_svr/README.html)
- [AuTerm](https://github.com/thedjnK/AuTerm)
