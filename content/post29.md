+++
title = "UEFI Rust Std is now in Nightly"
description = "Initial support for UEFI Rust Std has been merged into upstream Rust"
date = "2023-09-25T11:45:12+05:30"

[taxonomies]
tags = ["rust", "uefi"]
+++
Hello everyone. I started working on Rust Std for UEFI last year as a part of my Google Summer of Code 2022 project under Tianocore. My mentors for this project were Michael Kinney and Michael Kubacki. After a year of work, initial support for UEFI Rust std has been merged into upstream. In this post, I will review what is currently possible and my plans.

<!-- more -->

# Introduction
Unified Extensible Firmware Interface (UEFI) is a specification for a software program that connects a computer's firmware to its operating system (OS). Work in this space is predominantly performed using C. Tianocore EDKII is the reference implementation of UEFI in C.

Contrary to people's conceptions, UEFI provides a lot of OS primitives (in the form of Windows-like protocols) like allocator, stdio, networking, etc. It is possible to run quite complex applications in UEFI, like memory checks, disk checks, etc. While writing UEFI applications in basic C is possible, you need to use something like EDKII to go beyond Hello World. EDKII is a large, complex environment with its own build system and thus might only be suitable for some projects.

On the other hand, GNU-EFI is a set of libraries and headers for compiling UEFI applications with a system's native GCC. However, you then need to convert the resulting ELF into UEFI-compatible PE. Since I have never used it personally, I cannot comment on how well it works.

Thus, having Rust std support would make it possible to write complex UEFI applications and utilities using standard Rust primitives. Since `rustc` allows cross-compilation, writing UEFI applications in Rust is as simple as creating a cargo project and compiling it for the `*-unknown-uefi` target. This makes Rust + Std one of the most straightforward development setup for UEFI applications

The current Rust support is limited but should provide a good start for anyone considering using Rust. I will also continue working on getting more Std functionality supported for UEFI.

# Current Status
The following Rust Std functionality is currently implemented:
1. **alloc**: This means all heap types should work.
2. **os_str**: OsString uses UTF-16 and UCS-2 under the hood, allowing more straightforward conversion to and from Rust UTF-8 Strings.

Additionally, some UEFI-specific functionality is provided under `os::uefi`:
1. **env**: This module provides access to the underlying SystemTable and ImageHandle.
    a. `system_table() -> NonNull<c_void>`
    b. `image_handle() -> NonNull<c_void>`
    c. `boot_services() -> Option<NonNull<c_void>>`
2. **ffi**: This module provides convenience methods for OsStrings (aka UTF-16 Strings). This is just a re-export of [`windows::ffi`](https://doc.rust-lang.org/std/os/windows/ffi/index.html).

Since the underlying SystemTable can be accessed, using this alongside the existing Rust UEFI ecosystem should be possible.

# Demo
Using the Standard library, let us create a small "Hello World" application.
1. Install Rust nightly. The build should be `2023-09-24` or later
```sh
rustup toolchain install nightly
```
2. Create a new cargo project:
```sh
cargo new hello_world
```
3. Populate `src/main.rs`.
```rust
#![feature(uefi_std)]

use r_efi::efi;
use std::os::uefi::ffi::OsStrExt;
use std::{ffi::OsString, panic};

fn main() {
    uefi_hello(std::os::uefi::env::system_table().as_ptr() as *mut _);
}

fn uefi_hello(st: *mut efi::SystemTable) {
    let s = String::from("Hello World From Rust\n");
    let s: OsString = s.into();
    let mut s: Vec<u16> = s.encode_wide().collect();
    s.push(0);
    let r =
        unsafe { ((*(*st).con_out).output_string)((*st).con_out, s.as_ptr() as *mut efi::Char16) };
    if r.is_error() {
        panic!("Failed to print")
    }
}
```
4. Build the binary
```sh
cargo +nightly build --target x86_64-unknown-uefi
```
5. Test. I am using OVMF in Qemu for this. To automate the process, I have written a simple [python script](https://github.com/Ayush1325/scripts/blob/main/Tianocore/RunQemu.py).
```sh
python RunQemu.py target/x86_64-unknown-uefi/debug/hello_world.efi
```

# Future
During my GSoC work period, I implemented a lot more of the standard library functionality ([GSoC22 Conclusion Post](@/post15.md)). So, I will work on slowly getting those parts merged. It might be slow since I do it in my free time, but feel free to help if you are interested.

There might be some bugs and rough edges right now. It would be great if people could test it out and suggest areas of improvement.

Consider [supporting me](@/pages/supportme.md) if you like my work.
