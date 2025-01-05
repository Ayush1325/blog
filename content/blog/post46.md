+++
title = "My Week in Code #15"
description = "Updates regarding BeagleBoard Rust Imager"
date = "2025-01-05T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "rust"]
+++

Happy New Year, everyone! 🎉 2024 was a fantastic year for my coding journey, and I’m thrilled to carry that energy into 2025.

Let's jump into the weekly updates and kick off another exciting year of coding adventures. 🚀

# BeagleBoard Rust Imager

A lot of work went into the BeagleBoard Rust Imager this week. I still haven't figured out Macos completely, but at least other platforms are moving along well. At this point, I will probably release v1.0 without MacOS support since there is not much I can do about it.

## CLI Improvements

Most of the past work concentrated on improving the GUI and making it as user-friendly as possible. However, as a terminal enjoyer, I prefer to use the CLI when possible. In lieu of that, I spent most of the past week bringing up the CLI to the point of being great to use, at least on Linux. Let us now go over some of the work:

### Allow post-flash customization

The GUI has an option to perform some customization (e.g., setting user name and password, setting wifi ssid and password, etc.) after flashing the SD card. This ability in CLI would allow much easier automation when one needs to flash multiple devices. So, the CLI is now able to perform all the customization options available in the GUI.

### Support for remote os images

Since it might be useful to be able to flash OS images from the internet, I ended up adding an option to flash remote OS images. I am a bit unsure regarding this one since one could just use `wget`, but it might end up being useful once I add an option to browse remote images using the default `distros.json` that the GUI uses.

### Improved Help

One area that saw massive improvements is the help messages for the CLI (along with all the subcommands). It should be fairly simple for people to understand how to use the CLI now.

### Shell Completion

The CLI now supports generating shell completions at both runtime and compile-time. I am using [clap_complete](https://crates.io/crates/clap_complete) to achieve this. The compile-time generation is done using [xtask](@/blog/post46.md#xtask) instead of `build.rs`, since it allows me to generate the completion on demand.

### Manpage

I am also generating manpage(s) for the CLI and all of its subcommands at compile time using [clap_mangen](https://crates.io/crates/clap_mangen). Similar to shell completion, I am using [xtask](@/blog/post46.md#xtask) for this.

### Xtask

While working on shell completion and manpage support, I came across [cargo xtask spec](https://github.com/matklad/cargo-xtask), which allows adding free-form automation to a Rust project. It's pretty neat since it does not need anything extra (other than cargo and rustc) and gives the power of full-fat Rust. It's especially nice for cases like shell completion and manpage generation since they need to interact with Rust code and cannot be done with just `Makefile recipe`.

My current policy will be to use xtask when I need something that requires bash scripting but will stick to `Makefile` when just calling POSIX utilities is sufficient for the job. I have seen projects use xtask for CI, but I still prefer `Makefile` for that kind of functionality.

### Showcase

Here are some showcases for the CLI.

#### Home Help

```shell
❯ bb-imager-cli --help
A streamlined tool for creating, flashing, and managing OS images for BeagleBoard devices.

Usage: bb-imager-cli [OPTIONS] <COMMAND>

Commands:
  flash                Command to flash an image to a specific destination
  list-destinations    Command to list available destinations for flashing based on the selected target
  format               Command to format SD Card
  generate-completion  Command to generate shell completion
  help                 Print this message or the help of the given subcommand(s)

Options:
      --quiet    Suppress standard output messages for a quieter experience
  -h, --help     Print help
  -V, --version  Print version
```

#### Flashing SD Card Help

```shell
❯ bb-imager-cli flash sd --help
Flash an SD card with customizable settings for BeagleBoard devices
Usage: bb-imager-cli flash <DST> sd [OPTIONS]

Options:
      --no-verify                      Disable checksum verification post-flash
      --hostname <HOSTNAME>            Set a custom hostname for the device (e.g., "beaglebone")
      --timezone <TIMEZONE>            Set the timezone for the device (e.g., "America/New_York")
      --keymap <KEYMAP>                Set the keyboard layout/keymap (e.g., "us" for the US layout)
      --user-name <USER_NAME>          Set a username for the default user. Requires `user_password`.
                                       Required to enter GUI session due to regulatory requirements.
      --user-password <USER_PASSWORD>  Set a password for the default user. Requires `user_name`.
                                       Required to enter GUI session due to regulatory requirements.
      --wifi-ssid <WIFI_SSID>          Configure a Wi-Fi SSID for network access. Requires `wifi_password`
      --wifi-password <WIFI_PASSWORD>  Set the password for the specified Wi-Fi SSID. Requires `wifi_ssid`
  -h, --help                           Print help
```

#### Flashing Remote image

```shell
❯ bb-imager-cli flash --image-remote $IMG_URL --image-sha256 $IMG_SHA256 /dev/ttyACM0 bcf
[1] Preparing
[2] Verifying    [█████████████████████████████████████████████████████████████████████████████████████████████████████████████] [100 %]
[3] Flashing     [█████████████████████████████████████████████████████████████████████████████████████████████████████████████] [100 %]
[4] Verifying
```

#### Flashing Local image

```shell
❯ bb-imager-cli --quite flash $DESTINATION $IMG_PATH /dev/ttyACM0 bcf
```

## GUI Improvements

While putting the GUI alongside the Raspberry Pi Imager, I observed that my GUI and its elements, like buttons, were a lot bigger. After some discussion, I ended up deciding to match the rpi-imager size. So, the default window size is now 680x450, with the buttons and other UI elements scaled down accordingly.

I also added some rounded corners since they look nicer in the scaled-down UI. The buttons and text are still a bit bigger than the RPI-imager, but since it fits fine, I am keeping it like this.

### Home Screenshot

{{ image(src="/images/post46/home.webp") }}

## Future

I have been working on utilizing Gitlab issues as much as possible in the [bb-imager-rs](https://openbeagle.org/ayush1325/bb-imager-rs/-/issues) to make it easier for anyone wishing to start contributing. Hopefully, I can get some new contributors this way.

# Ending Thoughts

That is all for the week. Hopefully, 2025 will be a good year for me and [BeagleBoard.org](https://www.beagleboard.org/).

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs)
