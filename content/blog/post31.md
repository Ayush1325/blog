+++
title = "MicroBlocks on Beagleconnect Freedom"
description = "A brief introduction to use MicroBlocks on Beagleconnect Freedom"
date = "2024-06-15T00:30:12+05:30"

[taxonomies]
tags = ["beagleconnect_freedom", "zephyr", "arduino", "microblocks"]
+++

Hello everyone. I have been recently working on porting [MicroBlocks](https://microblocks.fun/) to [Beagleconnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom). This also involved work improving [Arduino module for Zephyr](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core) since [MicroBlocks](https://microblocks.fun/) uses Arduino APIs. I will go over using MicroBlocks on beagleconnect freedom.

<!-- more -->

# What is MicroBlocks?

[MicroBlocks](https://microblocks.fun/) is a free, Scratch-like blocks programming language for learning physical computing with educational microcontroller boards. It allows complete beginners to get started quickly, from children as young as nine years old up through adults of all ages.

However, MicroBlocks isn't just for beginners. It can be used to learn electronics, instrument science experiments, automate your home, and much more.

# Technical Details

Let us first review the technical details of how all of this works. Beagleconnect Freedom already has Zephyr support. However, MicroBlocks uses Arduino APIs, and migrating everything to Zephyr APIs would take a lot of work. Additionally, the final goal is to have everything merged upstream, and thus, it is better to have as little maintenance burden as possible.

I am using [Arduino Core API module for Zephyr](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core), which implements Arduino APIs on top of Zephyr. Adding support for Beagleconnect Freedom was pretty painless since all it amounts to is a device tree overlay ([board porting instructions](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core/blob/next/documentation/variants.md)). I had to add some missing APIs, but the process was pretty painless. Some missing APIs have already been merged, while others have active PRs.

# Building MicroBlocks Firmware

The latest MicroBlocks images for Beagleconnect Freedom are available in the [CI](https://openbeagle.org/ayush1325/zephyr/-/artifacts). I will now go over building the images locally for those interested.

## Install dependencies

Install dependencies present in [Getting Started with Zephyr](https://docs.zephyrproject.org/latest/develop/getting_started/index.html).

## Setup Zephyr Sdk

Setup Zephyr Sdk by following instructions in [Getting Started with Zephyr](https://docs.zephyrproject.org/latest/develop/getting_started/index.html).

## Get Zephyr and Python dependencies

1. Create a new virtual environment:

```bash
python3 -m venv ~/zephyrproject/.venv
```

2. Activate the virtual environment:

```bash
source ~/zephyrproject/.venv/bin/activate
```

3. Install west:

```bash
pip install west
```

4. Get the Zephyr source code. I am currently using my own fork, although all the required changes already have open PRs:

```bash
west init -m https://openbeagle.org/ayush1325/zephyr.git --mr microblocks ~/zephyrproject
cd ~/zephyrproject
west update
```

5. Export a Zephyr CMake package. This allows CMake to automatically load boilerplate code required for building Zephyr applications.

```bash
west zephyr-export
```

6. Zephyr’s scripts/requirements.txt file declares additional Python dependencies. Install them with pip.

```bash
pip install -r ~/zephyrproject/zephyr/scripts/requirements.txt
pip install cc1352-flasher
```

## Setup Arduino module for Zephyr

1. Clone Arduino APIs. These are licensed under GNU LGPL 2.1 and thus cannot be included in the Zephyr module.

```bash
git clone https://github.com/arduino/ArduinoCore-API.git ~/.ArduinoCore-API
```

2. Link ArduinoCore-APIs:

```bash
sed '/WCharacter.h/ s/./\/\/ &/' ~/.ArduinoCore-API/api/ArduinoAPI.h > ~/.ArduinoCore-API/api/tmpArduinoAPI.h
mv ~/.ArduinoCore-API/api/tmpArduinoAPI.h ~/.ArduinoCore-API/api/ArduinoAPI.h
ln -sf ~/.ArduinoCore-API/api ~/zephyrproject/modules/lib/Arduino-Zephyr-API/cores/arduino/.
```

## Build MicroBlocks image

1. Git clone

```bash
git clone https://openbeagle.org/ayush1325/smallvm.git -b zephyr ~/zephyrproject/
```

2. Build and flash Image

```bash
west build -b beagleconnect_freedom ~/zephyrproject/smallvm/zephyr -p && west flash
```

# Run MicroBlocks ide

1. Download MicroBlocks [standalone ide](https://microblocks.fun/download). The browser version of ide does not support manually selecting ports.

2. MicroBlocks ide should automatically detect Beagleconnect Freedom. It might take upto a minute.

It is also possible to manually select beagleconnect freedom tty by enabling `settings->show advanced blocks` in case you have lot of usb devices connected.

# Demo

Just running blink example on beagleconnect freedom

{{ image(src="/images/post31/microblocks_ide.webp")}}

# Future

The PRs for all the required changes are already open for [Arduino module for Zephyr](https://github.com/zephyrproject-rtos/gsoc-2022-arduino-core) and [Zephyr](https://www.zephyrproject.org/). I will also try to get support merged to upstream MicroBlocks which should allow much easier porting of MicroBlocks to Zephyr supported boards in the future.

I will also be writing more posts regarding Arduino API support for beagleconnect freedom, which is already in a pretty good state.

# Conclusion
There are still a lot of bugs, so feel free to open [issues](https://openbeagle.org/ayush1325/smallvm/-/issues). I will also write more articles regarding Arduino API support for Beagleconnect Freedom.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful Links
- [MicroBlocks](https://microblocks.fun/)
- [Issue Tracker](https://openbeagle.org/ayush1325/smallvm/-/issues)
- [Zephyr MicroBlocks Fork](https://openbeagle.org/ayush1325/zephyr/-/tree/microblocks?ref_type=heads)
- [Arduino module for Zephyr MicroBlocks Fork](https://github.com/Ayush1325/gsoc-2022-arduino-core/tree/microblocks)
- [Smallvm Zephyr Fork](https://openbeagle.org/ayush1325/smallvm/-/tree/zephyr?ref_type=heads)
