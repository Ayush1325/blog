+++
title = "Writing UEFI Protocol in Rust"
description = "A post about writing UEFI Protocol in Rust"
date = "2022-08-27T22:22:12+05:30"

[taxonomies]
tags = ["rust", "tianocore", "gsoc22", "uefi"]
+++
Hello everyone. As a part of Porting Rust std for UEFI, I had to write a hacky implementation of pipes (using UEFI Variables) for piping output from launched programs. However, this implementation had some problems and failed to run tests using the feature `panic_abort_tests`. Since I had some time this week, I decided to write a Protocol to fix the pipes. Along the way, I also found many ways to shoot myself on foot. So I decided to write this post documenting all the gotchas I discovered while writing a new protocol in Rust. This post will only contain 1 of the two protocols I decided to create, internally named `UEFI_COMMAND_PROTOCOL`.

<!-- more -->

**Note:** This Protocol is not public API and thus should not be relied upon/used outside of Rust std, at least for the foreseeable future.

# Protocol Definition
The `UEFI_COMMAND_PROTOCOL` has a straightforward use: to share the Handles of pipes (stdout, stderr, stdin) with the image to be started. It might contain more stuff in the future, but for right now, that is all it does. It is installed on all UEFI executables launched using the Rust `std::process::Command` API and needs Pipes (In case they don't need any kinds of Pipes, this Protocol will not be installed).

## Structure
Here is the Rust Structure to define the Protocol:
```rust
#[repr(C)]
pub struct Protocol {
    pub stdout: r_efi::efi::Handle,
    pub stderr: r_efi::efi::Handle,
    pub stdin: r_efi::efi::Handle,
}
```
As you can see, there is not much to it.

## Guid
Here is the protocol GUID:
```rust
pub const PROTOCOL_GUID: r_efi::efi::Guid = r_efi::efi::Guid::from_fields(
    0xc3cc5ede,
    0xb029,
    0x4daa,
    0xa5,
    0x5f,
    &[0x93, 0xf8, 0x82, 0x5b, 0x29, 0xe7],
);
```

<br>

# Protocol Usage
When constructing and installing a Protocol from Rust, some things must be kept in mind. Here are some of the lessons I learned the hard way:

## The Protocol should be non-movable
By default, all Rust variables are movable in memory. Once a UEFI Protocol is registered, it has to be valid until it is uninstalled. Installing a Protocol whose interface is on the stack can lead to all sorts of UB. In my particular case, I decided to use Heap allocated Protocol in `std::box::Box`. From what I understand, there are ways to move the contents of `Box` in memory. However, for my simple needs, it was sufficient since I was not performing any operations on Box once the Protocol was installed.

In practice, the code looks something like this:
```rust
let mut protocol = Box::new(uefi_command_protocol::new(stdout, stderr, stdin));
install_interface(&mut command handle, &mut protocol).unwrap();
// Do something with Command
```
For internal usage, I have wrappers that handle uninstalling the protocol on Drop to play nice with Rust memory management. It might be possible to have a better solution using rust `std::pin::Pin` wrapper, but I haven't used it personally.

## Use EFI_BOOT_SERVICES.InstallProtocolInterface() and EFI_BOOT_SERVICES.UninstallProtocolInterface()
Most modern UEFI drivers use the `EFI_BOOT_SERVICES.InstallMultipleProtocolInterfaces()` and `EFI_BOOT_SERVICES.UninstallMultipleProtocolInterfaces()` for installing and uninstalling protocols on a handle. However, `rustc` currently doesn't support variable arguments for UEFI. For more information about this, you can look at this [issue](https://github.com/r-efi/r-efi/issues/12).

## Taking care of lifetimes
You need to make sure to strictly define the lifetimes of protocols with their lifetimes as expected by UEFI. I wrote some structures in Rust std to make the internal working easier. However, having a difference in lifetimes can lead to everything ranging from CPU exceptions to an infinite loop.

Also take care of any data the protocol points to.

## Taking care of Rust Drop Order
There will be cases when you want the strictly define the drop order of members of Protocol wrapper structures. I found this great [answer](https://stackoverflow.com/questions/41053542/forcing-the-order-in-which-struct-fields-are-dropped) for this. However, the rule of thumb is that Rust, by default, drops members in the same order they were declared. So make sure to keep this in mind.

## Avoid storing Rust-specific types in Protocol
Storing dynamic data in UEFI Protocol is generally not a great idea. However, Rust-specific types (Vec, Box, etc) can lead to all kinds of UB since all Rust types are movable by default. I worked around this by storing pointers to a struct that wraps over the Rust-specific type. A simple example is the following:
```rust
struct PipeData {
    data: VecDequeue<u8>
}

#[repr(C)]
pub struct Protocol {
    data: *mut PipeData
}
```

<br>

# Conclusion
Since there are not many examples of Rust usage in UEFI drivers, there are not many standard practices established yet. With this post, I hope to help anyone trying to use Rust for UEFI driver development. Finally, with the new `UEFI_COMMAND_PROTOCOL` and `UEFI_PIPE_PROTOCOL`, Rust std port can now use complete `libtest` (even `should_panic`, which was previously broken). This is useful since even if you go the `no_std` route, you can still use std to test your `no_std` code. Feel free to check out [Rust std for UEFI](https://github.com/rust-lang/rust/pull/100316).

Consider [supporting me](@/pages/supportme.md) if you like my work.
