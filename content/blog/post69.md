+++
title = "PocketBeagle 2 Zephyr DFU"
description = "A demo of using DFU to load Zephyr firmware on PocketBeagle 2"
date = "2025-11-15T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "pocketbeagle2", "zephyr"]
+++

Hello everyone. It has been a while since my last post. Today, I will go over using DFU over USB on PocketBeagle 2. I will only be loading the firmware to RAM, since that should be the normal use case when doing Zephyr development. It is possible to do DFU to MMC or SD Card, so I might cover it in the future.

# Motivation

During normal Zephyr development for [PocketBeagle 2 A53](https://docs.zephyrproject.org/latest/boards/beagle/pocketbeagle_2/doc/index.html), one has to copy the firmware to the SD card (using some sort of MicroSD reader). This is fine for deployment, but it can become cumbersome when doing rapid changes.

Using DFU over USB allows bypassing the requirement of an SD Card, thus providing a much better setup for rapid prototyping. Using DFU, it is possible to go from scratch to a Zephyr application without having an SD Card plugged in, and with minimal user input.

# Working Demo

## Steps

1. Download the DFU u-boot binaries. I am using `v2025.10-am62-pocketbeagle2-11.02.04` for the demo.

```sh
wget https://github.com/beagleboard/u-boot-pocketbeagle2/releases/download/v2025.10-am62-pocketbeagle2-11.02.04/tiboot3-usbdfu.bin
wget https://github.com/beagleboard/u-boot-pocketbeagle2/releases/download/v2025.10-am62-pocketbeagle2-11.02.04/tispl.bin
wget https://github.com/beagleboard/u-boot-pocketbeagle2/releases/download/v2025.10-am62-pocketbeagle2-11.02.04/u-boot-zephyrdfu.img
```

2. Install [dfu-util](https://dfu-util.sourceforge.net/) CLI.

3. Plug in PocketBeagle 2 and check if it is available.

```sh
dfu-util -l
```

4. Upload `tiboot3-usbdfu.bin`:

```sh
dfu-util -R -a bootloader -D ./tiboot3-usbdfu.bin
```

5. Upload `tispl.bin`:

```sh
dfu-util -R -a tispl.bin -D ./tispl.bin
```

6. Upload `u-boot-zephyrdfu.img`:

```sh
dfu-util -R  -a u-boot.img -D ./u-boot-zephyrdfu.img
```

7. Upload `zephyr.bin`. I am using [MicroPython firmware](https://github.com/beagleboard/micropython-builder/releases/download/continuous-release/pocketbeagle_2-am6254-a53.bin.xz) here.

```sh
wget https://github.com/beagleboard/micropython-builder/releases/download/continuous-release/pocketbeagle_2-am6254-a53.bin.xz
xz -d pocketbeagle_2-am6254-a53.bin.xz
dfu-util -R  -a zephyr.bin -D pocketbeagle_2-am6254-a53.bin
```

## Logs

Here are the logs from both host and device to help with the process.

### Host

```sh
❯ wget https://github.com/beagleboard/u-boot-pocketbeagle2/releases/download/v2025.10-am62-pocketbeagle2-11.02.04/tiboot3-usbdfu.bin
                                                                                                                                       
HTTP response 302  [https://github.com/beagleboard/u-boot-pocketbeagle2/releases/download/v2025.10-am62-pocketbeagle2-11.02.04/tiboot3-?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-11-15T04%3A54%3A12Z&rscd=attachment%3B+filename%3Dtiboot3-usbdfu.bin&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-11-15T03%3A54%3A06Z&ske=2025-11-15T04%3A54%3A12Z&sks=b&skv=2018-11-09&sig=KFlqUGcd0puVvVrY31Sa%2BclfnTVG%2FlmvDDhCjlC5Ssc%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MzE4MDU0NCwibmJmIjot-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-11-15T03%3A54%3A06Z&ske=2025-11-15T04%3A54%3A12Z&sks=b&skv=2018-11-09&sig=KFlqUGcd0puVvVrY31Sa%2BclfnTVG%2FlmvDDhCjlC5Ssc%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MzE4MDU0NCwibmJmIjoSaving 'tiboot3-usbdfu.bin'                                                                                                            HTTP response 200  [https://release-assets.githubusercontent.com/github-production-release-asset/972828200/cf85b54e-eea3-4648-905f-c09fe970d4e1?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-11-15T04%3A54%3A12Z&rscd=attachment%3B+filename%3Dtiboot3-usbdfu.bin&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-11-15T03%3A54%3A06Z&ske=2025-11-15T04%3A54%3A12Z&sks=b&skv=2018-11-09&sig=KFlqUGcd0puVvVrY31Sa%2BclfnTVG%2FlmvDDhCjlC5Ssc%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MzE4MDU0NCwtiboot3-usbdfu.bin   100% [====================================================================================>]  304.80K  165.75KB/si

❯ wget https://github.com/beagleboard/u-boot-pocketbeagle2/releases/download/v2025.10-am62-pocketbeagle2-11.02.04/tispl.bin
                                                                                                                                       
HTTP response 302  [https://github.com/beagleboard/u-boot-pocketbeagle2/releases/download/v2025.10-am62-pocketbeagle2-11.02.04/tispl.bi?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-11-15T04%3A53%3A50Z&rscd=attachment%3B+filename%3Dtispl.bin&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-11-15T03%3A53%3A28Z&ske=2025-11-15T04%3A53%3A50Z&sks=b&skv=2018-11-09&sig=kSXoqxRhMuW3y9Hw40CAJKevVgyjWtyQDwbZ1RQ5czI%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MzE4MDU1OSwibmJmIjoxNzYzMTgwMjU5skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-11-15T03%3A53%3A28Z&ske=2025-11-15T04%3A53%3A50Z&sks=b&skv=2018-11-09&sig=kSXoqxRhMuW3y9Hw40CAJKevVgyjWtyQDwbZ1RQ5czI%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MzE4MDU1OSwibmJmIjoxNzYzMTgwMjU5Saving 'tispl.bin'                                                                                                                     HTTP response 200  [https://release-assets.githubusercontent.com/github-production-release-asset/972828200/8d57a55b-5bd2-4b9d-b391-909781bda7b8?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-11-15T04%3A53%3A50Z&rscd=attachment%3B+filename%3Dtispl.bin&rsct=application%2Foctet
-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-11-15T03%3A53%3A28Z&ske=2025-11-15T04%3A53%3A50Z&sks=b&skv=2018-11-09&sig=kSXoqxRhMuW3y9Hw40CAJKevVgyjWtyQDwbZ1RQ5czI%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MzE4MDU1OSwibmJmIjoxNzYztispl.bin            100% [====================================================================================>]  924.72K  376.88KB/se-content-disposition=attac[Files: 1  Bytes: 924.72K [145.76KB/s] Redirects: 1  Todo: 0  Errors: 0               ]
                                                                                                                                       
❯ wget https://github.com/beagleboard/u-boot-pocketbeagle2/releases/download/v2025.10-am62-pocketbeagle2-11.02.04/u-boot-zephyrdfu.img
                                                                                                                                       
HTTP response 302  [https://github.com/beagleboard/u-boot-pocketbeagle2/releases/download/v2025.10-am62-pocketbeagle2-11.02.04/u-boot-zAdding URL: https://release-assets.githubusercontent.com/github-production-release-asset/972828200/73c288ae-6ed1-4ca4-b1d8-8e2b8ff7a479?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-11-15T04%3A54%3A51Z&rscd=attachment%3B+filename%3Du-boot-zephyrdfu.img&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-11-15T03%3A54%3A12Z&ske=2025-11-15T04%3A54%3A51Z&sks=b&skv=2018-11-09&sig=V%2FIQP9oBpZQ658Q1onytLpGWaYJ4BuKYNYSr%2FShfJyI%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MzE4MDU2NywibmJmIAdding URL: https://release-assets.githubusercontent.com/github-production-release-asset/972828200/73c288ae-6ed1-4ca4-b1d8-8e2b8ff7a479?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-11-15T04%3A54%3A51Z&rscd=attachment%3B+filename%3Du-boot-zephyrdfu.img&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-11-15T03%3A54%3A12Z&ske=2025-11-15T04%3A54%3A51Z&sks=b&skv=2018-11-09&sig=V%2FIQP9oBpZQ658Q1onytLpGWaYJ4BuKYNYSr%2FShfJyI%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MzE4MDU2NywibmJmISaving 'u-boot-zephyrdfu.img'HTTP response 200  [https://release-assets.githubusercontent.com/github-production-release-asset/972828200/73c288ae-6ed1-4ca4-b1d8-8e2b8ff7a479?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-11-15T04%3A54%3A51Z&rscd=attachment%3B+filename%3Du-boot-zephyrdfu.img&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-11-15T03%3A54%3A12Z&ske=2025-11-15T04%3A54%3A51Z&sks=b&skv=2018-11-09&sig=V%2FIQP9oBpZQ658Q1onytLpGWaYJ4BuKYNYSr%2FShfJyI%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MzE4MDU2Nu-boot-zephyrdfu.img 100% [====================================================================================>]  875.71K  327.64KB/sDXwtXH4&response-content-di[Files: 1  Bytes: 875.71K [195.34KB/s] Redirects: 1  Todo: 0  Errors: 0               ]tion%2Foctet-stream]

❯ sudo dfu-util -l
dfu-util 0.11

Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
Copyright 2010-2021 Tormod Volden and Stefan Schmidt
This program is Free Software and has ABSOLUTELY NO WARRANTY
Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

Found DFU: [0451:6165] ver=0200, devnum=11, cfg=1, intf=0, path="3-1", alt=1, name="SocId", serial="01.00.00.00"
Found DFU: [0451:6165] ver=0200, devnum=11, cfg=1, intf=0, path="3-1", alt=0, name="bootloader", serial="01.00.00.00"
 
❯ sudo dfu-util -R -a bootloader -D ./tiboot3-usbdfu.bin
dfu-util 0.11

Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
Copyright 2010-2021 Tormod Volden and Stefan Schmidt
This program is Free Software and has ABSOLUTELY NO WARRANTY
Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

dfu-util: Warning: Invalid DFU suffix signature
dfu-util: A valid DFU suffix will be required in a future dfu-util release
Opening DFU capable USB device...
Device ID 0451:6165
Device DFU version 0110
Claiming USB DFU Interface...
Setting Alternate Interface #0 ...
Determining device status...
DFU state(2) = dfuIDLE, status(0) = No error condition is present
DFU mode device DFU version 0110
Device returned transfer size 512
Copying data from PC to DFU device
Download        [=========================] 100%       312122 bytes
Download done.
DFU state(6) = dfuMANIFEST-SYNC, status(0) = No error condition is present
dfu-util: unable to read DFU status after completion (LIBUSB_ERROR_IO)

❯ sudo dfu-util -R -a tispl.bin -D ./tispl.bin
dfu-util 0.11

Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
Copyright 2010-2021 Tormod Volden and Stefan Schmidt
This program is Free Software and has ABSOLUTELY NO WARRANTY
Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

dfu-util: Warning: Invalid DFU suffix signature
dfu-util: A valid DFU suffix will be required in a future dfu-util release
Opening DFU capable USB device...
Device ID 0451:6165
Device DFU version 0110
Claiming USB DFU Interface...
Setting Alternate Interface #0 ...
Determining device status...
DFU state(2) = dfuIDLE, status(0) = No error condition is present
DFU mode device DFU version 0110
Device returned transfer size 4096
Copying data from PC to DFU device
Download        [=========================] 100%       946923 bytes
Download done.
DFU state(7) = dfuMANIFEST, status(0) = No error condition is present
DFU state(2) = dfuIDLE, status(0) = No error condition is present
Done!
Resetting USB to switch back to Run-Time mode

❯ sudo dfu-util -R  -a u-boot.img -D ./u-boot-zephyrdfu.img
dfu-util 0.11

Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
Copyright 2010-2021 Tormod Volden and Stefan Schmidt
This program is Free Software and has ABSOLUTELY NO WARRANTY
Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

dfu-util: Warning: Invalid DFU suffix signature
dfu-util: A valid DFU suffix will be required in a future dfu-util release
Opening DFU capable USB device...
Device ID 0451:6165
Device DFU version 0110
Claiming USB DFU Interface...
Setting Alternate Interface #1 ...
Determining device status...
DFU state(2) = dfuIDLE, status(0) = No error condition is present
DFU mode device DFU version 0110
Device returned transfer size 4096
Copying data from PC to DFU device
Download        [=========================] 100%       896735 bytes
Download done.
DFU state(7) = dfuMANIFEST, status(0) = No error condition is present
DFU state(2) = dfuIDLE, status(0) = No error condition is present
Done!
Resetting USB to switch back to Run-Time mode

❯ sudo dfu-util -R  -a zephyr.bin -D pocketbeagle_2-am6254-a53.bin
dfu-util 0.11

Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
Copyright 2010-2021 Tormod Volden and Stefan Schmidt
This program is Free Software and has ABSOLUTELY NO WARRANTY
Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

dfu-util: Warning: Invalid DFU suffix signature
dfu-util: A valid DFU suffix will be required in a future dfu-util release
Opening DFU capable USB device...
Device ID 0451:6165
Device DFU version 0110
Claiming USB DFU Interface...
Setting Alternate Interface #0 ...
Determining device status...
DFU state(2) = dfuIDLE, status(0) = No error condition is present
DFU mode device DFU version 0110
Device returned transfer size 4096
Copying data from PC to DFU device
Download        [=========================] 100%       459192 bytes
Download done.
DFU state(7) = dfuMANIFEST, status(0) = No error condition is present
DFU state(2) = dfuIDLE, status(0) = No error condition is present
Done!
Resetting USB to switch back to Run-Time mode
```

### PocketBeagle 2 UART

```sh
❯ tio /dev/ttyACM0
[09:56:45.642] tio 3.9
[09:56:45.642] Press ctrl-t q to quit
[09:56:45.643] Connected to /dev/ttyACM0
�
 U-Boot SPL 2025.10-gebc75819ab98 (Nov 11 2025 - 19:33:39 +0000)
SYSFW ABI: 4.0 (firmware rev 0x000b '11.2.3--v11.02.03 (Fancy Rat)')
Changed A53 CPU frequency to 1400000000Hz (T grade) in DT
SPL initial stack usage: 13424 bytes
Trying to boot from DFU
dwc3-am62 dwc3-usb@f900000: unable to get ti,syscon-phy-pll-refclk regmap
####DOWNLOAD ... OK
Ctrl+C to exit ...
Authentication passed
Authentication passed
Authentication passed
Loading Environment from nowhere... OK
init_env from device 10 not supported!
Authentication passed
Authentication passed
Starting ATF on ARM64 core...

NOTICE:  BL31: v2.12.8(release):lts-v2.12.8
NOTICE:  BL31: Built : 19:32:42, Nov 11 2025
I/TC:
I/TC: OP-TEE version: 4.8.0 (gcc version 13.3.0 (Ubuntu 13.3.0-6ubuntu2~24.04)) #1 Tue Nov 11 19:33:32 UTC 2025 aarch64
I/TC: WARNING: This OP-TEE configuration might be insecure!
I/TC: WARNING: Please check https://optee.readthedocs.io/en/latest/architecture/porting_guidelines.html
I/TC: Primary CPU initializing
I/TC: GIC redistributor base address not provided
I/TC: Assuming default GIC group status and modifier
I/TC: SYSFW ABI: 4.0 (firmware rev 0x000b '11.2.3--v11.02.03 (Fancy Rat)')
I/TC: HUK Initialized
I/TC: Disabling output console

U-Boot SPL 2025.10-gebc75819ab98 (Nov 11 2025 - 19:34:58 +0000)
SYSFW ABI: 4.0 (firmware rev 0x000b '11.2.3--v11.02.03 (Fancy Rat)')
SPL initial stack usage: 2048 bytes
Trying to boot from DFU
dwc3-am62 dwc3-usb@f900000: unable to get ti,syscon-phy-pll-refclk regmap
####DOWNLOAD ... OK
Ctrl+C to exit ...
Authentication passed
Authentication passed


U-Boot 2025.10-gebc75819ab98 (Nov 11 2025 - 19:35:36 +0000)

SoC:   AM62X SR1.0 HS-FS
Reset reason: POR
Model: BeagleBoard.org PocketBeagle2
DRAM:  448 MiB (total 512 MiB)
Core:  88 devices, 31 uclasses, devicetree: separate
MMC:   mmc@fa10000: 0, mmc@fa00000: 1
Loading Environment from nowhere... OK
In:    serial@2860000
Out:   serial@2860000
Err:   serial@2860000
Net:   No ethernet found.
Press SPACE to abort autoboot in 0 seconds
##DOWNLOAD ... OK
Ctrl+C to exit ...
## Starting application at 0x80200000 ...
*** Booting Zephyr OS build 8a72d776c5af ***
Secondary CPU core 1 (MPID:0x1) is up
Secondary CPU core 2 (MPID:0x2) is up
Secondary CPU core 3 (MPID:0x3) is up
MicroPython 9999553aae on 2025-10-20; zephyr-pocketbeagle_2 with am6254
Type "help()" for more information.
>>>
```

# Bonus

I have added initial support for DFU in [bb-imager](https://github.com/beagleboard/bb-imager-rs). While the GUI support is not ready yet, it is possible to use the bb-imager-cli.

```sh
bb-imager-cli flash dfu "03:01:0451:6165" \
	"bootloader" ./tiboot3-usbdfu.bin \
	"tispl.bin" ./tispl.bin \
	"u-boot.img" ./u-boot-zephyrdfu.img \
	"zephyr.bin" ./pocketbeagle_2-am6254-a53.bin
```

Here, `03:01:0451:6165` is the device identifier. This can be obtained by using the command `bb-imager-cli list-destinations dfu`. The identifier might feel a bit awkward, but it seems to be the most stable way to identify a DFU device in a cross-platform manner, since libusb does not expose any path.

# Ending Thoughts

That is all for this post. Hopefully, this serves as an example for anyone trying DFU on PocketBeagle 2.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [PocketBeagle 2](https://www.beagleboard.org/boards/pocketbeagle-2)
- [dfu-util](https://dfu-util.sourceforge.net/)
- [TI Docs](https://software-dl.ti.com/processor-sdk-linux/esd/AM62X/latest/exports/docs/linux/Foundational_Components/U-Boot/UG-DFU.html)
