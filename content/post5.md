+++
title = "Google Summer of Code 2022"
description = "A brief introduction to my project"
date = "2022-06-14"

[taxonomies]
tags = ["rust", "tianocore", "gsoc22"]
+++

Hello everyone, I am Ayush Singh, a second-year student at the Indian Institute of Technology, Dhanbad, India. As a part of Google Summer of Code 2022, I will be working on Adding Rust Support for building UEFI Applications and Modules under the Tianocore organization. In this post, I will describe this project's goals and set up my development environment.

<!-- more -->

You can follow my GSoC-related blog posts using this [feed](https://www.programmershideaway.xyz/tags/gsoc22/atom.xml). I will currently be working under my personal [fork](https://github.com/Ayush1325/rust). In the future, it might be moved into the [edk2-staging](https://github.com/tianocore/edk2-staging) repository. The final goal is to have the support merged upstream. However, that might not be immediately possible.

<br>

# Project Goals
The current project goals, as discussed with the mentors, are given below:
1. Get most of Rust-std running under the UEFI target
2. Add testing support for UEFI target
3. Pass all the general tests of `library/std`
4. Possibly add support for ARM and RISCV UEFI targets (currently, only X86_64, AARCH64, and I686 are present)

<br>

# Setting up Development Environment
## Setup Rust
### Install Rustup
I will be using rustup to install and manage Rust since I will be working with multiple versions of standard libraries and compilers.
```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
It will ask for some options, but I just set it to default.

### Install Rust toolchains
I will be installing stable as well as nightly toolchains.
For Stable:
```sh
rustup toolchain install stable
```
For Nightly:
```sh
rustup toolchain install nightly
```

Finally, add `~/.cargo/bin` to `PATH`. Alternatively, we can just source `$HOME/.cargo/env`.

<br>

## Setup rust project
### Setup Git Repository
I will mostly follow the [Getting Started](https://rustc-dev-guide.rust-lang.org/getting-started.html) section of the rustc-dev-guide for this. First, we need to clone the `rust-lang/rust` repository. I prefer using ssh:
```sh
git clone git@github.com:rust-lang/rust.git
```
Next, I will add my personal fork as a remote:
```sh
git remote add personal git@github.com:Ayush1325/rust.git
```

### Install depenencies
Rust repository contains a script (`x.py`) to help us install most of the dependencies we need and configures the bootstrapping process. However, since we need to compile to a different target, we need to build `rust-lld`, which requires extra dependencies. Since I am using Fedora, I will list the command I used to install dependencies:
```sh
sudo dnf install ccache cmake python ninja-build llvm-devel llvm-libunwind-devel zlib-devel lld clang clang-tools-extra qemu
```

Next, we will use the script to configure our build.
```sh
./x.py setup
```
Since I will mostly work on `library/std`, I select the library option during configuration. We will need to modify the generated `config.toml` since we will need `rust-lld` for cross-compilation. Here is my config.toml:
```toml
# Includes one of the default files in src/bootstrap/defaults
profile = "library"
changelog-seen = 2

[llvm]
download-ci-llvm = "if-available"
ccache = true

[rust]
lld = true
incremental = true
```

<br>

## Build toolchain
Now, we will build the local toolchain using the following command:
```sh
./x.py build --stage 1
```
We build the stage1 compiler since we will need to use this compiler to build sample applications for testing (Since we cannot use the testing framework yet).

Finally, we will add this newly built toolchain using rustup:
```sh
rustup toolchain link stage1 build/x86_64-unknown-linux-gnu/stage1
```

Now we can use this toolchain to build any rust project.

<br>

# Build a UEFI hello_world
Now to check that our new toolchain works, we will build a simple hello world application (no_std).
## Create a binary application
```sh
cargo new hello_world
```

## Configure Cargo
First, we need to add the `r-efi` crate to `Cargo.toml` as follows:
```toml
[dependencies]
r-efi = "4.0"
```
Next, we will add a file `.cargo/config.toml` to configure building `core` and `alloc`. This way we do not need to specify `build-std` option at the commmand line.
```toml
[unstable]
build-std = ["core", "compiler_builtins"]
build-std-features = ["compiler-builtins-mem"]
```
Finally, we will set the current project to use our local toolchain:
```
rustup override set stage1
```

## Main Code
Here is the code for `src/main.rs` file:
```rust
#![no_main]
#![no_std]

use r_efi::efi;

#[panic_handler]
fn panic_handler(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[export_name = "efi_main"]
pub extern "C" fn main(_h: efi::Handle, st: *mut efi::SystemTable) -> efi::Status {
    let s = [
        0x0048u16, 0x0065u16, 0x006cu16, 0x006cu16, 0x006fu16, // "Hello"
        0x0020u16, //                                             " "
        0x0057u16, 0x006fu16, 0x0072u16, 0x006cu16, 0x0064u16, // "World"
        0x0021u16, //                                             "!"
        0x000au16, //                                             "\n"
        0x0000u16, //                                             NUL
    ];

    // Print "Hello World!".
    let r =
        unsafe { ((*(*st).con_out).output_string)((*st).con_out, s.as_ptr() as *mut efi::Char16) };
    if r.is_error() {
        return r;
    }

    // Wait for key input, by waiting on the `wait_for_key` event hook.
    let r = unsafe {
        let mut x: usize = 0;
        ((*(*st).boot_services).wait_for_event)(1, &mut (*(*st).con_in).wait_for_key, &mut x)
    };
    if r.is_error() {
        return r;
    }

    efi::Status::SUCCESS
}
```

## Build and Test application
We can build the application using a normal `cargo build` now. To test the application, we will use qemu. To test out EFI applications in qemu, I have written a small python [script](https://github.com/Ayush1325/scripts/blob/main/Tianocore/RunQemu.py). The `hello_world.efi` file that should be inside `target/x86_64-unknown-uefi/debug/`. Here is the output screenshot:
{{ image(src="/images/post5/run_qemu.webp")}}

<br>

# Conclusion
With this, we can start implementing parts of std for UEFI and testing them in actual applications. Feel free to ask/make any suggestions on the Tianocore mailing list.

<br>

# Important Links
1. [GSoC22 feed](https://www.programmershideaway.xyz/tags/gsoc22/atom.xml)
2. [Current Work Repository](https://github.com/Ayush1325/rust)
