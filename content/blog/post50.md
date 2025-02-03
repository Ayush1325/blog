+++
title = "My Week in Code #19"
description = "Updates regarding Devicetree, PocketBeagle and Rust std for UEFI"
date = "2025-02-02T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "beagleplay", "linux", "rust", "pocketbeagle2", "uefi"]
+++

Hello everyone. Since [PocketBeagle 2](https://www.beagleboard.org/boards/pocketbeagle-2) has launched, most of the development was focused on it. Let's go over everything.

# Enable Greybus BeaglePlay driver in defconfig

As per suggestion of Nishanth Menon, I have added gb_beagleplay to the defconfig. [Here](https://lore.kernel.org/all/20250131-defconfig-beagleplay-v3-1-f556b851ff96@beagleboard.org/) is the latest version of the patch.

# Export Symbols RFC

As alluded to in my prior posts, I have been working on using `export-symbols` in devicetree to add support for add-on connectors such as MikroBUS. This was first proposed by Herve Codina in the [following Linux kernel patches](https://lore.kernel.org/all/20241209151830.95723-1-herve.codina@bootlin.com/). Since then, I have sent patches to add support to [devicetree compiler](https://lore.kernel.org/all/20250110-export-symbols-v1-1-b6213fcd6c82@beagleboard.org/) and [fdtoverlay](https://lore.kernel.org/devicetree-compiler/86a7a08c-d81c-43d4-99fb-d0c4e9777601@beagleboard.org/T/#t).

The overall goal is to make export-symbols part of devicetree specification instead of just a Linux specific extension. This is to allow sharing of add-on board overlays with ZephyrRTOS, which has boards like [BeagleConnect Freedom](https://www.beagleboard.org/boards/beagleconnect-freedom) with support for MikroBUS.

To that end, I have sent an [RFC](https://lore.kernel.org/devicetree-spec/024685af-4694-4c22-bf83-a04dde6e32bd@beagleboard.org/T/#u) in the devicetree specification mailing list to see if anyone has any objections/flaws to point out. I will be sending a patch adding the node to the specification in a week or 2, if there are no objections.

# Pocketbeagle 2

[Pocketbeagle 2](https://www.beagleboard.org/boards/pocketbeagle-2) is now available for purchase. While, everything is mostly working now, I have been working on adding a few things to just make the support as good as possible.


## Debug Port

In the early phases of development, most people were using [PocketBeagle® TechLab](https://www.beagleboard.org/boards/techlab) with PocketBeagle 2, and thus using the UART exposed by it. However, PocketBeagle 2 has a debug UART (compatible with Raspberry Pi Debug Probe) on board.

There were some issues to get U-Boot working with this UART, but other than U-Boot R5 logs, everything is redirected to use the Debug Port. We are actively working on getting R5 to also use the debug port, but it should not affect general usage, since the boot selection menu is shown by A53 core.

## Examples

In a recent discussion, it was also decided to add examples for beginners to get started quickly. Initially, I was going to use Python, but given that Beagle Boards seem to attract a lot of people in Kernel and U-Boot space, it was decided that I should go ahead with writing examples in Rust.

With Rust support in Linux kernel finally reaching a good enough state, this means the same language can be used for both userspace IoT projects and Kernel space drivers.

Rust is also quite popular nowadays, so it might help attract new potential contributors. The examples can be found [here](https://openbeagle.org/ayush1325/vsx-examples). Hopefully, they will start being shipped in default images, and will be accessible with the Theia local Theia editor.

Feel free to create issues regarding the examples you would like to be added.

### Hello Beagle

Simple example to test that Rust has been set up correctly.

#### Usage

```sh
debian@pocketbeagle2:~/vsx-examples/PocketBeagle2$ cargo run -p hello_beagle
   Compiling hello_beagle v0.1.0 (/home/debian/vsx-examples/PocketBeagle2/hello_beagle)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 5.64s
     Running `target/debug/hello_beagle`
Hello, World! From PocketBeagle2
```

### EEPROM

Read and parse EEPROM contents in Rust.

#### Usage

```sh
debian@pocketbeagle2:~/vsx-examples/PocketBeagle2$ cargo run -p eeprom
   Compiling eeprom v0.1.0 (/home/debian/vsx-examples/PocketBeagle2/eeprom)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 3.99s
     Running `target/debug/eeprom`
EEPROM Data: Contents {
    magic_number: [
        170,
        85,
        51,
        238,
    ],
    hdr1: Header {
        id: 1,
        len: 55,
    },
    hdr2: Header {
        id: 16,
        len: 46,
    },
    board_info: BoardInfo {
        name: "POCKETBEAGL2A00G",
        version: "A0",
        proc_number: "0000",
        variant: "0G",
        pcb_revision: "0G",
        schematic_bom_revision: "A0",
        software_revision: "00",
        vendor_id: "01",
        build_week: "34",
        build_year: "24",
        serial: "PB20000001",
    },
    hdr3: Header {
        id: 17,
        len: 2,
    },
    ddr_info: 4776,
    termination: 254,
}
```

### Blinky

Simple example to toggle GPIOs. Uses Pin `P1_20` by default.

Also provides a simple helper function to find PIN by name.

#### Usage

```sh
debian@pocketbeagle2:~/vsx-examples/PocketBeagle2$ cargo run -p blinky
   Compiling blinky v0.1.0 (/home/debian/vsx-examples/PocketBeagle2/blinky)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 3.50s
     Running `target/debug/blinky`
ON
OFF
ON
OFF
..
```

## MSPM0 Firmware

PocketBeagle 2 contains an MSPM0, which is used as ADC and EEPROM. The MSPM0 firmware can be found [here](https://openbeagle.org/pocketbeagle/mspm0-adc-eeprom/-/releases).

Here are the instructions to flash MSPM0:

1. Download the firmware:

```sh
wget https://openbeagle.org/api/v4/projects/249/packages/generic/mspm0_adc_eeprom/0.13/mspm0_adc_eeprom.txt
```

2. Download [bb-imager-rs](https://openbeagle.org/ayush1325/bb-imager-rs/-/releases) latest version. Currently, that is v0.0.3:

```sh
wget https://openbeagle.org/api/v4/projects/832/packages/generic/bb-imager-cli/0.0.3/bb-imager-cli-aarch64-unknown-linux-gnu.tar.xz
tar -xvf bb-imager-cli-aarch64-unknown-linux-gnu.tar.xz
cd usr/bin
```

3. Flash

```sh
./bb-imager-cli flash pb2 --img-local /path/to/firmware --no-eeprom
```

It is also possible to flash any other custom firmware in Ti-TXT or iHex format.

# Rust std for UEFI

I have been having an issue with building and testing toolchain reliably. [Here](https://rust-lang.zulipchat.com/#narrow/channel/122651-general/topic/Cross.20Compiling.20std.20stops.20working) is the link to Zulip discussion regarding it. However, I have yet to find a solution, so things might get delayed a bit. Still, there is a lot of backlog still left to merge, so things should be fine for a while.

## Process Args

While trying to add environment variable support to [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html), I found out that args support was broken. The [PR](https://github.com/rust-lang/rust/pull/136186) to fix it has now been merged.

## Process Env

Std process has been missing environment variables till now. This is because environment variable support was missing in general at the time I add support for process. However, since environment variable support has been [merged](https://github.com/rust-lang/rust/pull/127462), I have created a [PR](https://github.com/rust-lang/rust/pull/136418) to add environment variables to [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html).

The implementation closely resembles [UEFI Shell Execute](https://github.com/tianocore/edk2/blob/2d2642f4832ebc45cb7d5ba9430b933d953b94f2/ShellPkg/Application/Shell/ShellProtocol.c#L1700), which basically applies any environment changes before executing the binary, and rolls back the changes after execution concludes.

## Path support

Support for UEFI paths has now been [merged](https://github.com/rust-lang/rust/pull/135475), this makes it much easier to construct absolute paths, from relative as well as UEFI shell mappings.

It also implements support for checking if the path is absolute, which should work correctly now thanks to an earlier [PR](https://github.com/rust-lang/rust/pull/135405).# Ending Thoughts

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [Pocketbeagle 2](https://www.beagleboard.org/boards/pocketbeagle-2)
- [Export Symbols RFC](https://lore.kernel.org/devicetree-spec/024685af-4694-4c22-bf83-a04dde6e32bd@beagleboard.org/T/#u)
- [PocketBeagle® TechLab](https://www.beagleboard.org/boards/techlab)
- [MSPM0 ADC EEPROM Firmware](https://openbeagle.org/pocketbeagle/mspm0-adc-eeprom)
- [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs)
