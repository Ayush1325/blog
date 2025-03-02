+++
title = "My Week in Code #23"
description = "Updates regarding PocketBeagle 2 examples, Devicetree Compiler and Specification"
date = "2025-03-02T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "pocketbeagle2", "rust", "devicetree"]
+++

Hello everyone. A typical work week. Let's go over everything.

# PocketBeagle 2 Examples

Continuing the work from [last week](@/blog/post53.md), I finished adding Python examples for all the prior Rust examples. I have also added a lot of new examples, mostly centered around the hardware available in [TechLab cape for PocketBeagle 2](https://www.beagleboard.org/boards/techlab).

## EEPROM

I already had a Rust example to parse Beagle EEPROM contents. A python version for the example has also been added. I have also introduced `BeagleEeprom` type in both the Rust and Python helper libraries to make it easier for other people to extend. I also ended up learning about [struct module in python](https://docs.python.org/3/library/struct.html), which ended up being better than I expected. I still prefer Rust and C for packed structs, but at least python has something.

Here is the Python example running:

```bash
debian@pocketbeagle2:~/vsx-examples/PocketBeagle-2/eeprom/python$ python main.py
BeagleEeprom(magic_number=(170, 85, 51, 238),
             hdr1=Header(hdr_id=1, length=55),
             hdr2=Header(hdr_id=16, length=46),
             board_info=BoardInfo(name='POCKETBEAGL2A00G',
                                  version='A0',
                                  proc_number='0000',
                                  variant='0G',
                                  pcb_revision='A0',
                                  schematic_bom_revision='A0',
                                  software_revision='00',
                                  vendor_id='01',
                                  build_week='02',
                                  build_year='25',
                                  serial='PB20000183'),
             hdr3=Header(hdr_id=17, length=2),
             ddr_info=4776,
             termination=254)
```

## Photo Diode

[TechLab cape](https://www.beagleboard.org/boards/techlab) has a Photo Diode attached to pin `P1.19`. I have added Python and Rust examples to use the Photo Diode to detect whether it is dark or light, based on a threshold. The helper library also contains `LightSensor` for general usage. The full details can be found in the following [PR](https://openbeagle.org/beagleboard/vsx-examples/-/merge_requests/9#4904e12bfa96bd96372ab6138efe4977fc157b4d).

It would be nice to write a Photo Diode kernel driver at some point, since currently the examples are directly reading ADC values.

## Seven Segments

[TechLab cape](https://www.beagleboard.org/boards/techlab) has two 7-segment displays. The displays are connected to `MCP23S18`, an IO extender, which is connected to PocketBeagle 2 over SPI. After a bit of grepping, I found that the upstream kernel already has a driver for [seven segment displays](https://elixir.bootlin.com/linux/v6.13.5/source/drivers/auxdisplay/seg-led-gpio.c), so I am using it instead of driving the GPIOs from userspace. The full details can be found in the following [PR](https://openbeagle.org/beagleboard/vsx-examples/-/merge_requests/10).

Currently, both the 7-segment displays are driven by separate drivers, since there does not exist a 14-segment GPIO driver for now. The subsystem seems to have support for 14 segment, so maybe it would be useful to write one.

## Tonal Buzzer

[TechLab cape](https://www.beagleboard.org/boards/techlab) has a buzzer connected to pin `P2.30`. Since the pin does not have hardware PWM, and producing notes only really needs 50% duty cycle at comparatively low frequencies, I am using the [pwm-gpio driver](https://elixir.bootlin.com/linux/v6.13.5/source/drivers/pwm/pwm-gpio.c). The full details can be found in the following [PR](https://openbeagle.org/beagleboard/vsx-examples/-/merge_requests/11).

The example currently plays Hedwig's theme from the Harry Potter Movies, which is ported from an [Arduino example](https://github.com/robsoncouto/arduino-songs/blob/master/harrypotter/harrypotter.ino). I might end up adding the buzzer as an alsa device in the future, but let's see how that goes.

## RGB LED

[TechLab cape](https://www.beagleboard.org/boards/techlab) has an RGB LED. I am using the [led-pwm-multicolor driver](https://elixir.bootlin.com/linux/v6.13.5/source/drivers/leds/rgb/leds-pwm-multicolor.c). The full details can be found in the following [PR](https://openbeagle.org/beagleboard/vsx-examples/-/merge_requests/13).

The example cycles through different hues.

# TechLab Cape Devicetree Overlay

The PocketBeagle 2 examples, specially when using TechLab Cape need a devicetree overlay for some specific pinmux and devices such 7-segments, RGB Led, etc. The overlay can be found in the following [PR](https://openbeagle.org/beagleboard/BeagleBoard-DeviceTrees/-/merge_requests/109). This overlay will be present in new distro images going forward.

# Devicetree Compiler support for /./

Normally, node properties in devicetree can only be replaced with new ones. A lot of hardware devicetrees actually include base devicetrees for SOCs, etc. However, to add some new values to properties, such as adding some extra clocks, pinumx, etc. would require copying all the values from the included devicetree. This could become inconvenient and can allow things to go out of sync.

A few months ago, I sent a [patch series](https://lore.kernel.org/devicetree-compiler/Z24nldCpXpoT7RaK@zatzit/T/#t) a few months ago to add capability to append properties instead of overriding them. In that patch series, there was discussion regarding something more flexible, with a popular option being able to use the old value to construct the new property value. This will allow a lot of usecases such as appending, pre-appending, duplicating, etc.

In practice, it looks as follows:

```dts
dts-v1/;

/ {
    str-prop = "0";
};

/ {
    /* append */
    str-prop = /./, "1";
};

/ {
    /* pre-append */
    str-prop = "1", /./;
};

/ {
    /* duplicate */
    str-prop = /./, /./;
};
```

The full patch series can be found [here](https://lore.kernel.org/r/20250301-previous-value-v1-0-71d612eb0ea9@beagleboard.org).

# Devicetree Specification export-symbols

I have been pursuing various possible methods to allow upstream drivers for connector add-on-board setups such as [MikroBUS](https://www.mikroe.com/mikrobus), [Beagle Capes](https://docs.beagleboard.org/boards/capes/index.html), etc. The current favorite seems to be export-symbols.

It was initially proposed by [Herve Codina](https://lore.kernel.org/all/20241209151830.95723-1-herve.codina@bootlin.com/) and seems to be quite useful even outside add-on-board setups. At the basic level, `export-symbols` provide a way to define symbols local to the node, and can be used by any overlay or devicetree include that uses the node as target.

This in turn allows creating overlays for add-on-boards using generic names for the nodes such SPI, I2C, etc. instead of knowing about the device it is being applied to. Thus, the same overlay can be used across multiple connectors in different devices.

The initial proposal was for Linux kernel, but since I would like to use the same setup in [ZephyrRTOS](https://www.zephyrproject.org/) as well, I have been trying to make `export-symbols` a standard devicetree node. I already had patch series to add support to [dtc](https://lore.kernel.org/all/20250110-export-symbols-v1-1-b6213fcd6c82@beagleboard.org/) and [fdtoverlay](https://lore.kernel.org/devicetree-compiler/86a7a08c-d81c-43d4-99fb-d0c4e9777601@beagleboard.org/T/#t) as a proof of concept.

The full patch for adding it to the devicetree secification can be found [here](https://lore.kernel.org/all/20250225-export-symbols-v1-1-693049e3e187@beagleboard.org/). Feel free to chime in if the topic interests you.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [PocketBeagle 2 Examples](https://openbeagle.org/beagleboard/vsx-examples)
- [BeagleBoard Devicetrees](https://openbeagle.org/beagleboard/BeagleBoard-DeviceTrees/-/tree/v6.12.x-Beagle?ref_type=heads)
- [Devicetree Specification export-symbols patch](https://lore.kernel.org/all/20250225-export-symbols-v1-1-693049e3e187@beagleboard.org/)
- [Devicetree compiler /./ patch](https://lore.kernel.org/devicetree-compiler/Z24nldCpXpoT7RaK@zatzit/T/#t)
