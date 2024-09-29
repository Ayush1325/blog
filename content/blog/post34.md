+++
title = "My Week in Code #3"
description = "Updates regarding bluetooth on BeagleConnect Freedom, BeagleY-AI Zephyr support, BeagleBoard Imager and MicroPython"
date = "2024-09-29T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "zephyr", "rust", "beagleconnect-freedom", "beagley-ai"]
+++


Hello everyone. This was one of the lighter weeks since I couldn't get much results. But after some thinking, it is also vital to write about failed endeavors in case I pick something back.

# Bluetooth support for BeagleConnect Freedom

I will outline everything I encountered while trying to get Bluetooth support for [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) on Zephyr. While I could not do it in the end, I hope this can be helpful in any future work on this.

Support for most other [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) capabilities is already present in upstream Zephyr, so Bluetooth is the last remaining major feature we are missing.

After exploring the Zephyr Bluetooth source code, I have come to 2 possible approaches. I will be outlining the benefits, effort required, etc.

## Binary Blob
It is possible to use binary blobs for HCI support on cc1352p7. As is evident from [Ti docs](https://dev.ti.com/tirex/explore/node?node=A__AEK1U73z6SVVx8nu76ziDg__com.ti.SIMPLELINK_CC13XX_CC26XX_SDK__BSEc4rl__LATEST&placeholder=true), there is already an implementation of HCI for cc13xx devices. The following steps need to be taken to integrate it with Zephyr:

1. Ensure that the internal structure of HCI does not conflict with Zephyr (I assume it might use Ti-RTOS).
2. Add the blob to [hal_ti](https://github.com/zephyrproject-rtos/hal_ti). West supports fetching [binary blobs](https://docs.zephyrproject.org/latest/contribute/bin_blobs.html).
3. Write an HCI driver (see [hci_esp32](https://github.com/zephyrproject-rtos/zephyr/blob/main/drivers/bluetooth/hci/hci_esp32.c)).


This approach requires minimal changes to the current Ti approach and should be reasonably straightforward for anyone with access to the cc13xx HCI source. Additionally, minimal maintenance is required from the Ti side since they already maintain the HCI source for Ti-RTOS.

This approach is already in use in [ESP32 boards](https://docs.zephyrproject.org/latest/boards/01space/esp32c3_042_oled/doc/index.html#prerequisites). Thus, it should be accepted with minimal resistance.

## Zephyr Software Link Layer

Zephyr also supports software-based link layers, which are supported by Nordic hardware. While it should be possible to directly interface with cc13xx radio APIs in ROM and use the software link layer, the current ble stack was created around Nordic hardware. This means that supporting the Ti APIs would require significant time and effort to adapt things for Ti driverlib APIs.

I feel this approach only makes sense if Ti intends to officially support Zephyr since it will also require a permanent maintainer.

# BeagleY-AI Zephyr Support

During the [Beagle-Y-AI Hacking Sessions](https://discord.gg/AGU2VXE3?event=1283128283000606751), we went over how to get started with Zephyr on R5 core of BeagleY-AI. Here is the current [Zephyr repo](https://openbeagle.org/beagley-ai/zephyr/zephyr).

Check out the video for more details.

{{ youtube(id="0FvyT3ax-S0") }}

# BeagleConnect Freedom MicroPython

I have added basic [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) support to [draft PR](https://github.com/micropython/micropython/pull/15891). It currently contains support for the following:
- GPIO
- I2C
- SPI
- FLASH

I will move the support for this to the main branch once support for [Zephyr 0.37.0](https://github.com/micropython/micropython/pull/9335) is merged.

I am waiting on [upstream Zephyr PR](https://github.com/zephyrproject-rtos/zephyr/pull/78667) before adding support for ADC, PWM and IEEE802154g.

# BeagleBoard Imager Rust Utility

More updates for Rust port of BeagleBoard imager utility. I am thinking of creating a release soon.

## Support for more firmware file types

We are trying to make it possible to do development for [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) using [Ti Code Composer Studio](https://www.ti.com/tool/CCSTUDIO). However, the current flasher (both GUI and CLI) only supports flashing `.bin`. So to make things more ergonomic, I am planning to add support for [Ti-TXT](https://downloads.ti.com/docs/esd/SPRUI03/ti-txt-hex-format-ti-txt-option-stdz0795656.html), [Intel Hex](https://www.intel.com/content/www/us/en/programmable/quartushelp/17.0/reference/glossary/def_hexfile.htm), and other common formats.

I am using [bin_file](https://crates.io/crates/bin_file) to handle and parse the file formats. However, it was missing some functionality I needed. Here is the [PR](https://gitlab.com/robert.ernst.paf/bin_file/-/merge_requests/2).

I have already enabled support for new formats for flashing MSP430. Similar support for [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) is coming soon.

## Updated Look

The GUI look lines up with the current [BeagleBoard imager utility](https://www.beagleboard.org/bb-imager), complete with the background and font.

I use a window manager (sway), so the screenshots have no window decorations.

### Screenshots

Here are some screenshots from the application.

#### Home Page
{{ image(src="/images/post34/bbimager_home.webp")}}

#### Extra Configuration Page
{{ image(src="/images/post34/bbimager_config.webp")}}

#### Flashing Page
{{ image(src="/images/post34/bbimager_flash.webp")}}

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [MicroPython upstream Zephyr PR](https://github.com/micropython/micropython/pull/15891)
- [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom)
- [BeagleY-AI](https://www.beagleboard.org/boards/beagley-ai)
