+++
title = "My Week in Code #17"
description = "Updates regarding BeagleBoard Rust Imager and Rust fs support for UEFI"
date = "2025-01-19T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "rust", "pocketbeagle2", "uefi"]
+++

Hello everyone. A typical week for development. Let's go over everything.

# Zephyr Ti Hal Updates

A [few weeks ago](@/blog/post45.md), I created a PR to update the Simplelink SDK version in [hal_ti](https://github.com/zephyrproject-rtos/hal_ti) from 4.40.04.04 to 7.41.00.17. Now that it has been merged, hopefully, I can start working on adding Bluetooth support using the BLE5 Stack in the SDK.

# BeagleBoard Rust Imager

More work went into the BeagleBoard Rust imager this week. I hope to get the v1.0.0 release out by the end of this month and replace the old bb-imager.

## Introducing bb-imager-service

As mentioned [last week](@/blog/post46.md), I was searching for how to allow PocketBeagle2 MSPM0 firmware update from the GUI. To reiterate the problem, sysfs entries for [Firmware Upload API](https://docs.kernel.org/driver-api/firmware/fw_upload.html) is owned by root user and thus cannot be opened from the GUI without some kind of privilege escalation mechanism. As a reference, in case of SD card flashing, I am using UDisk2 D-Bus APIs to open the SD card, which internally uses polkit for privileged management.

After some googling, I came across the following [stack overflow answer](https://stackoverflow.com/questions/77516167/how-to-write-file-using-policykit-to-get-privilege):

- You could, of course, write your own D-Bus service which does that and install the D-Bus .service config to have it run as a system service with root privileges. (Try to keep the operations as limited and fine-grained as is reasonable; e.g. "enable global mode foo" – not "write arbitrary data to arbitrary file".)
- Failing that, one common method is to spawn `pkexec` to perform operations, which is… really just su with a nicer password prompt. Note that some distributions have recently removed `pkexec`.
- If your app is allowed to use either GNOME's GLib or KDE's KIO libraries, then both of those already have PolicyKit integration built-in for privileged file updates – the app could open admin:///etc/foo and GLib or KIO would elevate as needed.

Since I could not find much regarding the 3rd approach, and the 2nd approach seemed to be on the way out, I ended up going with a custom D-Bus service. I will probably need similar functionality for BeaglePlay cc1352p7, BeagleY-AI, etc., so it is not a bad idea to have a custom d-bus service.

Using [zbus](https://docs.rs/zbus/latest/zbus/) was pretty nice, and I have a better understanding of Linux systems, so it ended up being quite a learning experience. I am building a Debian package for the service for now since it is only supposed to be used in PocketBeagle2 right now. It also contains systemd service which activates on D-Bus connection.

The following articles were a lot of help in figuring everything out:

- [Register dbus service](https://nyirog.medium.com/register-dbus-service-f923dfca9f1)
- [Creating a D-Bus Service with dbus-python and Polkit Authentication](https://vwangsf.medium.com/creating-a-d-bus-service-with-dbus-python-and-polkit-authentication-4acc9bc5ed29)

## Overhaul config format

The config format I initially started with differed from RPI-imagers since I thought it would be more efficient. However, after adding a lot of boards and flasher types, I realized that every flashing target is not really a board. For example, having different board entries for `PocketBeagle2` and `PocketBeagle2 MSPM0` seemed weird.

While browsing [rpi-imager](https://github.com/raspberrypi/rpi-imager) last week, I found out that it seems to support bootloader updates in different rpi versions. The GUI presentation seemed nice, so after some deliberation, I adopted a format compatible with the RPI-imager's format. Since the `os_list` is a self-referential structure in the RPI-imager's config, it is possible to have a lot of sub-menus in the image selection menu, which is quite nice.

To elaborate, now there are only base boards (e.g., `PocketBeagle2`) in the Board Selection menu, and the Image Selection menu will have a sub-menu for `MSPM0 Firmware` support. 

Check out the [PR](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/53) for implementation details.

## Live destinations refresh

Similar to RPI-imager, destination refresh is now automatic using [subscriptions](https://docs.rs/iced/latest/iced/application/struct.Application.html#method.subscription). Check out the [PR](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/45) for implementation details.

## Better defaults in Extra Configuration

The application now queries the user's current system for the default suggestions regarding username, keyboard layout, timezone, etc. Check out the [PR](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/47) for details.

## Include udev rules

I have added [udev rules](https://openbeagle.org/ayush1325/bb-imager-rs/-/blob/main/bb-imager-gui/assets/packages/linux/udev/10-beagle.rules) to the Linux package of GUI to allow flashing BeagleConnect Freedom without needing superuser privileges. They also make development easier and allow using commands like `tio` without root.

# Rust std for UEFI

Since the PRs from the last post ([is_absolute refactoring](https://github.com/rust-lang/rust/pull/135405) and [OwnedDevicePath](https://github.com/rust-lang/rust/pull/135393)) have been merged, I have created a [PR](https://github.com/rust-lang/rust/pull/135475) to implement support for `std::path` for UEFI. Hopefully, it will be merged soon, making way for future PRs for `std::fs` support for UEFI.

# Ending Thoughts

That is all for the week. Hopefully, this series will keep people updated about my work and attract potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Rust Imager](https://openbeagle.org/ayush1325/bb-imager-rs)
- [Rust std for UEFI PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
