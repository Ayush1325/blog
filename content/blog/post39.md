+++
title = "My Week in Code #8"
description = "Updates regarding BeagleConnect Freedom, BeagleBoard Rust Imager and Rust std for UEFI"
date = "2024-11-03T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "beagleconnect-freedom", "beagleplay", "linux", "rust", "uefi"]
+++

Hello everyone. Another light week here. Let's go over everything.

# BeaglePlay PWM Patch

A while ago, I discovered that the MikroBUS PWM pin was not enabled in the upstream devicetree for BeaglePlay. So I created a [patch](https://lore.kernel.org/all/173021674666.3859929.3387135730934868690.b4-ty@ti.com/) to enable PWM and it was finally merged last week.

# BeagleConnect Freedom Debugging

Continuing the work from [last week](@/blog/post38.md), I was able to load Zephyr firmware that exhibited the stalling behavior. However, I still haven't been able to pinpoint the exact cause. You can follow the discussion in [Ti E2E](https://e2e.ti.com/support/wireless-connectivity/sub-1-ghz-group/sub-1-ghz/f/sub-1-ghz-forum/1430241/cc1352p7-beagleconnect-freedom-freezes-if-both-pwm-and-ieee802154-subg-is-used-together/)

I have created a [minimal example](https://github.com/Ayush1325/bcf-stall) to reproduce the problem.

# BeagleBoard Rust Imager

There has been some interest in making the Rust flasher the official flasher. To achieve that, there has been a lot of development towards v1.0.0.

## MacOS dmg support

Thanks to help from [Zain](https://openbeagle.org/superchamp234), I was able to create [MacOS dmg in CI](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/23). While most of the flashing functionality does not work on MacOS yet, having easy experimental CI builds should help development to at least ensure everything builds.

## Make bb-imager-rs Ready for crates.io

While exploring ways to create a Debian package, I stumbled across [debcargo](https://salsa.debian.org/rust-team/debcargo/), which is the official way to package Rust applications for Debian. Since it can potentially allow official packages in the future, I decided to prefer it over [cargo-deb](https://github.com/kornelski/cargo-deb). However, since debcargo builds the package from [crates.io](https://crates.io/), I needed to cleanup the out of tree patches for [rs-drivelist](https://github.com/ir1keren/rs-drivelist) and [bin_file](https://gitlab.com/robert.ernst.paf/bin_file). So, I moved the unmerged/unreleased code into bb-imager itself for now.

I still need to set up CI to push to crates.io automatically, but I am a bit unsure regarding how to handle the API key securely.

## Sd Card Formatting Support

I have added support for [SD card formatting](https://openbeagle.org/ayush1325/bb-imager-rs/-/merge_requests/24) since it is helpful for an imaging utility. However, it does not work on Windows yet.

## Performance Enhancements and Code improvements

Here is the list of minor improvements that occurred during the week:

- More responsive UI
- Make the default window size smaller
- Better handling of big os image lists.
- Code improvements to make contribution easier.

# File I/O Support for Rust std for UEFI

As eluded to [last week](@/blog/post38.md), File I/O PR is now ready for prime time. It supports the following kinds of paths:

- Relative Paths
- Absolute UEFI Shell Mapping
- Absolute UEFI Device Path

For now, I have only added support for files (so no directory APIs) to make review easier. However, once the file support is merged, directory API implementations will be a simple matter.

Currently, there is a lack of reviewers for UEFI Rust std PRs. So, if you have the experience, feel free to check out and/or review the [PR](https://github.com/rust-lang/rust/pull/129700).

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [BeagleBoard Imager Rust Port](https://openbeagle.org/ayush1325/bb-imager-rs)
- [My Rust std for UEFI open PRs](https://github.com/rust-lang/rust/pulls/Ayush1325)
