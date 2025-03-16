+++
title = "My Week in Code #25"
description = "Updates regarding PocketBeagle 2 examples, BeagleBoard Rust Imager and devicetree compiler"
date = "2025-03-16T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux", "pocketbeagle2", "rust", "devicetree"]
+++

Hello everyone. A typical work week. Let's go over everything.

# BeagleBoard Rust Imager

I am gearing up for a v1 release of [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs) soon. As such, a lot of work went into it this week.

## Justfiles

I have been using Makefiles for all the building and packaging tasks in [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs). Back when I started using makefiles, I did consider some alternatives:

1. [cargo xtask](https://github.com/matklad/cargo-xtask): I am using xtasks for things which require interacing with Rust code (for example generating manpage using [clap_mangen](https://crates.io/crates/clap_mangen)). However, it did not seem like a great fit for a general-purpose task runner since it is somewhat verbose to invoke shell commands. Additionally, compiling Rust isn't exactly the fastest, especially the first compilation which is the case in CI.
2. [just](https://just.systems/): Did not seem to be better enough than make, especially since make is pre-installed in most unix systems.

Additionally, the initial plan was to keep makefiles somewhat short, so readability was not among the top priorities. I was also much less familiar with `make` back then, and thus less aware of its pitfalls.

However, with time, the `make` based system grew quite a bit. Since makefile recipes did not have support for real arguments, a lot of recipes were based on pattern matching, which while being clever, ended up being difficult to understand and document. Additionally, while I did find some hacks to auto-generate help pages using a combination of `sed`, it was clunky and did not work as expected. There were also a lot of variables in Makefile that could change the build process in interesting ways, but none of them were documented.

Finally, I was essentially using `make` for running shell commands (aka tasks) and not as a build system to generate files. This meant that essentially, all makefile recipes were `PHONY` (see [this](https://stackoverflow.com/questions/2145590/what-is-the-purpose-of-phony-in-a-makefile) if you do not know what `PHONY` refers to in make).

It was somewhat clear that I needed to rewrite the makefile-based system from scratch, with good discoverability. So I thought, might as well re-evaluate the alternatives. In the end, I chose [just](https://just.systems/) because of its simplicity and comment-based documenting (similar to Rust). Additionally, since it is written in Rust, I can contribute any new features or fix bugs much more easily.

The porting was surprisingly simpler, and overall, the justfiles look much more maintainable. Additionally, anyone new to the project can easily get a handle on the build system now:

```sh
❯ just
Available recipes:
    default                                        # default recipe to display help information
    generate-cli-manpage                           # Generate Manpage for CLI
    generate-shell-completion SHELL                # Generate Shell completion for CLI

    [build]
    build-cli TARGET=_HOST_TARGET                  # Release build for Beagle Board Imager CLI. Should be used for final testing
    build-gui TARGET=_HOST_TARGET                  # Release build for Beagle Board Imager GUI. Should be used for final testing
    build-service TARGET=_HOST_TARGET              # Release build for Beagle Board Imager GUI. Should be used for final testing

    [housekeeping]
    check                                          # Run code checks
    clean                                          # Clean Artifacts
    test                                           # Run tests on workspace

    [packaging]
    package-cli-darwin-zip TARGET=_HOST_TARGET     # Create compressed CLI package for macOS
    package-cli-linux-xz TARGET=_HOST_TARGET       # Create compressed CLI package for Linux
    package-cli-windows-zip TARGET=_HOST_TARGET    # Create compressed CLI package for Windows
    package-gui-darwin-dmg TARGET=_HOST_TARGET     # Create macOS dmg package
    package-gui-linux-appimage TARGET=_HOST_TARGET # Create Appimage package for GUI
    package-gui-linux-deb TARGET=_HOST_TARGET      # Create debian package for GUI
    package-gui-windows-zip TARGET=_HOST_TARGET    # Create Windows portable zip package
    package-service-linux-deb TARGET=_HOST_TARGET  # Create debian package for Beagle Board Imager Service
    package-service-linux-xz TARGET=_HOST_TARGET   # Create compressed Beagle Board Imager service package for Linux
    release TARGET=_HOST_TARGET                    # Generate all supported packages for the target

    [run]
    run-cli FLAGS=''                               # Run CLI for quick testing on host. Flags can be used to modify the build
    run-gui FLAGS=''                               # Run GUI for quick testing on host. Flags can be used to modify the build

    [setup]
    setup-appimage INSTALL_DIR=_EXE_DIR            # Setup for building appimages
    setup-deb                                      # Setup for building debian packages
    setup-dmg                                      # Setup for building dmg
    setup-rust                                     # Setup rust toolcahin

❯ just --variables
APPIMAGETOOL PB2_MSPM0 RUST_BUILDER VERSION
```

I do have some problems with `just`, namely there does not seem to be a way to have conditional dependencies (I was able to do this in `make` by using some clever hacks). Additionally, I have not been able to get syntax highlight to work in `neovim`. So will try to fix those in the future. Overall, I am quite happy with [the migration](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/65).

## Loading Spinners

Since config (and its children) can be remote JSON files, which need to be downloaded, I have been hoping to add loading spinner screens for some much-needed visual oomph.

I was hoping for some third-party widget library with a loading spinner, but I could not find any such crate. Instead, I found an [example](https://github.com/iced-rs/iced/tree/master/examples/loading_spinners). I have added it as a crate to the workspace for now, and it seems to be working well. However, I probably should clean up the code at some point.

Checkout the [PR](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/67) for the exact diffs.

## Persistent Customization

Raspberry Pi imaging utility has the ability to save any customization settings applied by the user. This is useful since a lot of people using the GUI would probably prefer having the same settings be used for most of their SD card flashing. As such, I have added persistence to the customization settings, along with options to reset and abort the customizations.

The settings are stored in the platform-specific app config directories (`$HOME/.config/imagingutility/config.json` for Linux) in JSON format. While it is not intended to be modified by the end user, it is possible to do so fairly easily.

## Remove remote image support in CLI

I added the ability to download and flash images from URL in CLI in the last update. However, it ended up being somewhat limited since there are cases when one would not want to perform a sha256 check on the downloaded image. Additionally, the CLI interface required using non-positional arguments (`--img-local` and `--img-remote`) which felt a bit weird. 

Finally, it also seemed a bit unnecessary since anyone using CLI is probably already familiar with `wget`, and thus should use it for the downloading job. If there are any arguments regarding why remote image support would be useful, feel free to create an issue.

# PocketBeagle 2 Accelerometer Examples

[TechLab Cape](https://www.beagleboard.org/boards/techlab) contains an accelerometer, connected over I2C. I have added a simple Python and Rust example just reading the raw x, y, and z values from the accelerometer using iio sysfs entries. Checkout the [PR](https://openbeagle.org/beagleboard/vsx-examples/-/merge_requests/18) for full details.

In practice, it looks as follows:

```sh
debian@pocketbeagle2:~/vsx-examples/PocketBeagle-2/accelerometer/python$ python main.py
X =  -5 , Y =  -1 , Z =  258
X =  -4 , Y =  0 , Z =  258
X =  -5 , Y =  -1 , Z =  258
...
```

# Previous symbol dtc patches

I have sent the [2nd revisio](https://lore.kernel.org/all/20250311-previous-value-v2-0-e4a8611e956f@beagleboard.org/) of the previous symbol dtc patches. Here is the list of changes in this patch:

- Compare against expected dtb instead of checking each property individually in tests.
- Use xstrdup when copying refs so each marker has its own copy.
- Pass struct data by value, not by pointer.
- Add srcpos_free helper to properly free srcpos which have been extended previously.
- Remove open items since my original choices seem correct.

Let's wait for responses in the mailing list now.

# Target support in fdtoverlay

Linux kernel provides support to [apply devicetree overlays](https://docs.kernel.org/devicetree/kernel-api.html#c.of_overlay_fdt_apply) to specific nodes. This can allow drivers to dynamically apply and remove overlays, which can be used to add support for connector + addon board setups.

However, currently, there is no way to do the same thing with fdtoverlay. This means to change the target node for the overlay, the devicetree overlay source needs to be modified. This is somewhat inconvenient since any addon board overlays will be created assuming that they will be applied on the connector node directly, instead of the base tree. So the same overlays cannot be used with fdtoverlay.

Additionally, having support to specify the target node in fdtoverlay will also allow using the same overlays in static configurations for such connectors (since the overlay can be applied using u-boot).

In practice, it looks as follows:

base.dts:

```dts
/dts-v1/;

/ {
        my_node: my-node {
                prop = "hello";
        };
};
```

overlay.dtso:

```dts
/dts-v1/;
/plugin/;
/ {
	fragment@1 {
		target-path = "";
		__overlay__ {
			baz: baznode {
				baz-property = "baz";
			};
		};
	};
};
```

Apply overlay:

```sh
fdtoverlay -t "/my-node" -i base.dtb overlay.dtbo -o final.dtb
```

Result:

```dts
/ {
    my-node {
        prop = "hello";
        phandle = <0x00000001>;
        baznode {
            phandle = <0x00000002>;
            baz-property = "baz";
        };
    };
    __symbols__ {
        baz = "/baznode";
        my_node = "/my-node";
    };
};
```

It was a bit difficult to implement cleanly since libfdt does not allow allocations. However, it is working well now. Checkout the [patch series](https://lore.kernel.org/all/20250313-fdtoverlay-target-v1-0-dd5924e12bd3@beagleboard.org/) for more details. All that's left is to wait for someone to review the patches.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs)
- [PocketBeagle 2 Examples](https://openbeagle.org/beagleboard/vsx-examples)
- [Previous symbol dtc patches v2](https://lore.kernel.org/all/20250311-previous-value-v2-0-e4a8611e956f@beagleboard.org/)
- [Target support in fdtoverlay](https://lore.kernel.org/all/20250313-fdtoverlay-target-v1-0-dd5924e12bd3@beagleboard.org/)
