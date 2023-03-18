+++
title = "Use Restricted std in UEFI"
description = "Compile Rust std for UEFI"
date = "2022-06-16"

[taxonomies]
tags = ["rust", "tianocore", "gsoc22", "uefi"]
+++

Hello Everyone; in my last [post](@/post5.md), I set up the development environment for working on adding Rust support for UEFI. In this post, I will get a restricted version of std (basically a glorified core + alloc) to work for the x86_64 UEFI target. We will be starting with the `no_std` hello_world program from the last post.

<!-- more -->

# Add env module for UEFI in std
## Add uefi Module
First, I will add `uefi` module under `library/std/src/sys`. To do this, we will create a new file `library/std/src/sys/uefi/mod.rs`. We will point everything other than the `env` module to the unsupported module. The contents of `library/std/src/sys/uefi/mod.rs` are given below:
```rust
//! Platform-specific extensions to `std` for UEFI platforms.
//!
//! Provides access to platform-level information on UEFI platforms, and
//! exposes Unix-specific functions that would otherwise be inappropriate as
//! part of the core `std` library.
//!
//! It exposes more ways to deal with platform-specific strings ([`OsStr`],
//! [`OsString`]), allows to set permissions more granularly, extract low-level
//! file descriptors from files and sockets, and has platform-specific helpers
//! for spawning processes.
//!
//! [`OsStr`]: crate::ffi::OsStr
//! [`OsString`]: crate::ffi::OsString

#![deny(unsafe_op_in_unsafe_fn)]

#[path = "../unsupported/alloc.rs"]
pub mod alloc;
#[path = "../unsupported/args.rs"]
pub mod args;
#[path = "../unix/cmath.rs"]
pub mod cmath;
pub mod env;
#[path = "../unsupported/fs.rs"]
pub mod fs;
#[path = "../unsupported/io.rs"]
pub mod io;
#[path = "../unsupported/locks/mod.rs"]
pub mod locks;
#[path = "../unsupported/net.rs"]
pub mod net;
#[path = "../unsupported/os.rs"]
pub mod os;
#[path = "../windows/os_str.rs"]
pub mod os_str;
#[path = "../unix/path.rs"]
pub mod path;
#[path = "../unsupported/pipe.rs"]
pub mod pipe;
#[path = "../unsupported/process.rs"]
pub mod process;
#[path = "../unsupported/stdio.rs"]
pub mod stdio;
#[path = "../unsupported/thread.rs"]
pub mod thread;
#[path = "../unsupported/thread_local_key.rs"]
pub mod thread_local_key;
#[path = "../unsupported/time.rs"]
pub mod time;

#[path = "../unsupported/common.rs"]
#[deny(unsafe_op_in_unsafe_fn)]
mod common;
pub use common::*;
```
As you can see, `cmath`, `os_str`, and `path` point to the `unix` or `windows` module instead of `unsupported`. This is because `unsupported` does not provide any definition for those. The `cmath` unix module API can be provided by `compiler-builtins`, so that should work. However, `os_str` and `path` are simply placeholders for now.

## Implement env module
The env module is very basic and we just need to define a few constants for it. Here are the contents of `library/std/src/sys/uefi/env.rs`:
```rust
pub mod os {
    pub const FAMILY: &str = "";
    pub const OS: &str = "";
    pub const DLL_PREFIX: &str = "";
    pub const DLL_SUFFIX: &str = "";
    pub const DLL_EXTENSION: &str = "";
    pub const EXE_SUFFIX: &str = ".efi";
    pub const EXE_EXTENSION: &str = "efi";
}
```

## Export UEFI module contents
This can be done by adding the following lines to `library/std/sys/mod.rs`:
```rust
     } else if #[cfg(target_family = "wasm")] {
         mod wasm;
         pub use self::wasm::*;
+    } else if #[cfg(target_os = "uefi")] {
+        mod uefi;
+        pub use self::uefi::*;
     } else if #[cfg(all(target_vendor = "fortanix", target_env = "sgx"))] {
         mod sgx;
         pub use self::sgx::*;
```

And we now have the `env` module implemented. These constants are exposed under `std::env::consts`.

## Build the new toolchain
We will now need to build the toolchain again:
```sh
./x.py build --stage 1
```
stage1 should now be pointing to this new toolchain.

Now, we will try to print these constants to UEFI in a very primitive manner (since io, strings, etc have not been implemented yet).

<br>

# Compile hello_world with std
Firstly, we will remove the `no_std` attribute and `panic_handler` from `src/main.rs`. Then we will update `.cargo/config.toml` with the following contents:
```toml
[unstable]
build-std = ["std", "compiler_builtins"]
build-std-features = ["compiler-builtins-mem"]
```

On trying to compile the project now, we get the following error:
```sh
error[E0463]: can't find crate for `panic_abort`

error[E0658]: use of unstable library feature 'restricted_std'
  |
  = help: add `#![feature(restricted_std)]` to the crate attributes to enable

Some errors have detailed explanations: E0463, E0658.
For more information about an error, try `rustc --explain E0463`.
error: could not compile `hello_world` due to 2 previous errors
```

The first error is a [bug in build-std](https://github.com/rust-lang/rust/issues/83805#issuecomment-812874115). It doesn’t know how to handle panic_abort vs panic_unwind, so it doesn’t use either crate, resulting in the above error. It can be fixed by adding `panic_abort` to `.cargo/config.toml`. 

The second error can be fixed by adding the `restricted_std` feature to the `src/main.rs`.

Trying to build now and .......... It fails. We get the following error:
```sh
error: linking with `rust-lld` failed: exit status: 1
  |
  = note: "rust-lld" "-flavor" "link" "/NOLOGO" "/entry:efi_main" "/subsystem:efi_application" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/hello_world-bf96c91d419b97ff.3aumljakx58twtt1.rcgu.o" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/hello_world-bf96c91d419b97ff.11wx9iybsb4s2x54.rcgu.o" "/LIBPATH:/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps" "/LIBPATH:/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/debug/deps" "/LIBPATH:/var/home/ayush/Documents/Programming/Rust/rust/build/x86_64-unknown-linux-gnu/stage1/lib/rustlib/target/lib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/libr_efi-44f0e2c98ca397ed.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/libstd-0f82fcd1446bb823.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/libpanic_abort-6267bb336da2fa77.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/librustc_demangle-12ba4dfb1ce0974e.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/libstd_detect-af3466d26b0b584c.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/libhashbrown-282ed18a03cc159e.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/librustc_std_workspace_alloc-d47af0fe9b191470.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/libunwind-8ad628841136827a.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/libcfg_if-f793ff480fd551b6.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/liblibc-249d18e9ef84acfd.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/liballoc-4b1b3794d0343e8a.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/librustc_std_workspace_core-3a7d1ce70363f171.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/libcore-73e7a0474be04fb7.rlib" "/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/libcompiler_builtins-4f95060227077e02.rlib" "/NXCOMPAT" "/LIBPATH:/var/home/ayush/Documents/Programming/Rust/rust/build/x86_64-unknown-linux-gnu/stage1/lib/rustlib/target/lib" "/OUT:/var/home/ayush/Documents/Programming/Rust/uefi/hello_world/target/target/debug/deps/hello_world-bf96c91d419b97ff.efi" "/OPT:REF,NOICF" "/DEBUG" "/NODEFAULTLIB"
  = note: rust-lld: error: undefined symbol: __CxxFrameHandler3
          >>> referenced by libstd-0f82fcd1446bb823.rlib(std-0f82fcd1446bb823.std.ea9c30f6-cgu.2.rcgu.o):(.xdata)
          >>> referenced by libstd-0f82fcd1446bb823.rlib(std-0f82fcd1446bb823.std.ea9c30f6-cgu.2.rcgu.o):(.xdata)
          

error: could not compile `hello_world` due to previous error
```

This error is, well, a bit weird. We can fix the build for now by providing a blank implementation of `__CxxFrameHandler3`, but this needs more research. The following lines need to be added to `src/main.rs`:
```rust
#[no_mangle]
pub extern "C" fn __CxxFrameHandler3() {}
```

The application builds and runs fine now.

## Print EXE_EXTENSION constant
Now we will print EXE_EXTENSION to the console. Since we do not have io and string implemented yet, we will have to do it in a primitive way using `u16` arrays. The final `src/main.rs` is given below:
```rust
#![no_main]
#![feature(restricted_std)]

use r_efi::efi;
use std::env::consts;

#[no_mangle]
pub extern "C" fn __CxxFrameHandler3() {}

fn print_efi(s: &[u16], st: *mut efi::SystemTable) -> Result<(), r_efi::base::Status> {
    let r = unsafe { ((*(*st).con_out).output_string)((*st).con_out, s.as_ptr() as *mut efi::Char16) };

    if r.is_error() {
        Err(r)
    } else {
        Ok(())
    }
}

fn print_newline(st: *mut efi::SystemTable) -> Result<(), r_efi::base::Status> {
    let mut s = [0;2];
    create_const_uefi_str("\n", &mut s);
    print_efi(&s, st)
}

fn create_const_uefi_str(const_str: &str, uefi_array: &mut [u16]) {
    let mut i = 0;
    for v in const_str.bytes() {
        uefi_array[i] = v as u16;
        i += 1;
    }

    uefi_array[i] = 0x0000u16;
}

fn print_env_constants(st: *mut efi::SystemTable) -> Result<(), r_efi::base::Status> {
    let mut exe_extension_heading = [0; 16];
    create_const_uefi_str("exe_extension: ", &mut exe_extension_heading);
    print_efi(&exe_extension_heading, st)?;
    let mut exe_extension = [0; 5];
    create_const_uefi_str(consts::EXE_EXTENSION, &mut exe_extension);
    print_efi(&exe_extension, st)?;
    print_newline(st)
}

#[export_name = "efi_main"]
pub extern "C" fn main(_h: efi::Handle, st: *mut efi::SystemTable) -> efi::Status {
    let r = print_env_constants(st);

    if let Err(x) = r {
        return x;
    }

   efi::Status::SUCCESS
}
```

Here is the program running under qemu:

{{ image(src="/images/post6/run-qemu-1.webp")}}

With this, we are using our new `std` for UEFI. 

<br>

# Edit
The `__CxxFrameHandler3` blank implementation is no longer required in the `master` branch. I was previously basing my changes on the v1.61.0 tag. However, from now, I am going to work on master directly.

<br>

# Conclusion
Technically, we are now using `std` (even though none of it has yet been implemented). Now I will slowly start implementing parts of `std` starting with allocation. I also wanted to find a way to use the normal Rust `main` function instead of the current `efi_main`. However, this still does not seem possible (see [#29633](https://github.com/rust-lang/rust/issues/29633)). So, let's get allocation working and replace all the arrays with vectors in this code.

Consider [supporting me](@/pages/supportme.md) if you like my work.
