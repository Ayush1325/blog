+++
title = "UEFI Rust std has a new home"
description = "A short post to help external contribution"
date = "2022-08-05T20:44:12+05:30"

[taxonomies]
tags = ["rust", "tianocore", "gsoc22", "uefi"]
+++
Hello everyone. All my work on UEFI Rust std has been moved to [tianocore/rust](https://github.com/tianocore/rust). This should help with allowing more contribution from both Tianocore and Rust's sides. This post is just a sort of introduction to the new repository and how to get started.

<!-- more -->

# Documentation
The documentation of the std implementation for UEFI can be found at `src/doc/rustc/src/platform-support/unknown-uefi.md`. This can be built by using `x.py`:
```sh
./x.py doc src/doc/rustc/ --open
```
It outlines the requirements and the limitations of the current std for UEFI. It also contains a few examples to get users started.

# Development Workflow
The [tianocore/rust](https://github.com/tianocore/rust) repository accepts PRs. I will also accept patches in the edk2 mailing list (my email: [ayushdevel1325@gmail.com](ayushdevel1325@gmail.com)).

# Running Tests locally
Running Tests for UEFI is done using `remote-test-server` and `remote-test-client` as outlined in my [previous post](@/blog/post12.md). The only thing I would like to change from that guide is the command to run tests:
```sh
RUST_TEST_THREADS=1 TEST_DEVICE_ADDR="localhost:12345" ./x.py test src/test/ui/{FILE or Directory} --target x86_64-unknown-uefi --stage 1
```
Running tests in single-threaded, even on the host side, fixes a lot of tests that fail due to timeout.

# Conclusion
This post was just to spread the word about this work so more people can experiment and provide their feedback about it. 

Consider [supporting me](@/pages/about.md) if you like my work.
