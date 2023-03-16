+++
title = "Minimal Rust Std for UEFI"
description = "Summary of current state of Rust std for UEFI"
date = "2023-03-15T19:01:12+05:30"

[taxonomies]
categories = ["post"]
tags = ["rust", "uefi"]
+++

It has been quite a while since I gave any updates about Rust std implementation for UEFI. While the Google Summer of Code ended quite a bit ago (it's almost time for GSoC 23), I have been continuing my efforts to merge it upstream. Check out my [GSoC 22 Summary Post](@/post16.md) for a refresher on the project.

<!-- more -->

# Current State
Initially, I opened a [PR](https://github.com/rust-lang/rust/pull/100316) containing the complete implementation of Rust std for UEFI that I worked on for three months. However, it was somewhat incomplete and not thoroughly tested. Combining this with the massive size and the fact that it was a completely new addition to the std, it was pretty difficult to review. 

Initially, I split the parts unrelated to UEFI into separate PRs, which was somewhat helpful. However, after a suggestion from [David Rheinsberg](https://github.com/dvdhrm), I decided to create a stripped-down version of the main PR ([here](https://github.com/rust-lang/rust/pull/105861)), to get a very minimal std support merged upstream. This should help significantly simplify the review process and allows for maintaining high quality for everything merged.

The current plan is to get the minimal std merged and move forward from there. Let us now take a look at what this minimal PR contains.

# Minimal Rust Std for UEFI
The [current PR](https://github.com/rust-lang/rust/pull/105861) implements the following std modules for UEFI:
1. alloc: Based on [r-efi-alloc](https://github.com/r-efi/r-efi-alloc)
2. os_str: Based on WTF-8
3. env
4. math: Using [compiler-builtins](https://github.com/r-efi/r-efi-alloc)

While this might not seem like a lot, the main aim of the minimal std is to get the initial process of introducing a new std target to upstream Rust. Expanding the implementation once the initial version has already been upstreamed should be much easier.

The compiler takes care of generating the entry function for UEFI. This implementation allows just using the normal Rust main, as one would expect.

Finally, the minimal std also implements `panic-abort` to use `EFI_BOOT_SERVICES.Exit()`. However, since there is no Stdio support in this PR, it cannot print anything to stderr on panic.

# Example
This is just a small "Hello World" example to show the capabilites of the minimal std implementation. The project can be generated using `cargo` and the only dependency is [`r-efi`](https://crates.io/crates/r-efi). Here is the `main.rs` file:
```rust
use r_efi::{efi, protocols::simple_text_output};
use std::{
  ffi::OsString,
  os::uefi::{env, ffi::OsStrExt}
};

fn main() {
  let st = env::system_table().as_ptr() as *mut efi::SystemTable;
  // Allocate a OsString
  let s: OsString = OsString::from("Hello World!\n");
  let mut s: Vec<u16> = s.encode_wide().collect();
  s.push(0);
  let r =
      unsafe {
        let con_out: *mut simple_text_output::Protocol = (*st).con_out;
        let output_string: extern "efiapi" fn(_: *mut simple_text_output::Protocol, *mut u16) = (*con_out).output_string;
        output_string(con_out, s.as_ptr() as *mut efi::Char16)
      };
  assert!(!r.is_error())
}
```

# Conclusion
The Rust Std for UEFI is moving forward, albeit slower than I expected when I started. Nevertheless, I will try my best to see it to the end instead of dropping it abruptly. Feel free to check out the PR if this project interests you. Also, consider supporting me if you like my work.
