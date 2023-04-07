+++
title = "Handling ExitBootServices Event in Rust std"
description = "Describes the current implementation of handling ExitBootServices Event in Rust std implementation for UEFI"
date = "2023-04-07T16:07:12+05:30"

[taxonomies]
tags = ["rust", "uefi"]
+++
Hello everyone. Recently, there was a discussion about the exact scope/boundary of Rust std for UEFI. In this post, I am going to go over the different stages of UEFI and define what std for UEFI aims to target. Additionally, I will go over how the current implementation handles `ExitBootServices()`.

<!-- more -->

# Background
Unlike Operating Systems, UEFI is a collection of stages in the boot sequence, each with its own set of available resources/functionality. As such, it is vital to remember where in the UEFI boot stages a binary is supposed to be run. Here is a simple diagram of the different Boot Stages in UEFI.

{{ image(src="/images/post20/uefi_stages.webp")}}

Now I will go over each stage and their relation to Rust std.

## SEC – Security Phase
The Security (SEC) phase is the first phase in the PI Architecture and is responsible for the following:
- Handling all platform restart events
- Creating a temporary memory store
- Serving as the root of trust in the system
- Passing handoff information to the PEI Foundation

Rust std implementation should not be used here. In fact, even the `{arch}-unknown-uefi` target is not meant to be used in this phase.

## PEI – Pre-EFI Initialization
The PEI phase initially operates with the platform in a nascent state, leveraging only on-processor resources, such as the processor cache as a call stack, to dispatch Pre-EFI Initialization Modules (PEIMs). These PEIMs are responsible for the following:
- Initializing some permanent memory complement
- Describing the memory in Handoff Blocks (HOBs)
- Describing the firmware volume locations in HOBs
- Passing control into the Driver Execution Environment (DXE) phase.

Rust std implementation should not be used here. In fact, even the `{arch}-unknown-uefi` target is not meant to be used in this phase.

## DXE – Driver Execution Environment
There are several components in the DXE phase:
- DXE Foundation
- DXE Dispatcher
- A set of DXE Drivers

Rust std implementation can be used here.

## BDS – Boot Device Select
The BDS phase is responsible for the following:
- Initializing console devices
- Loading device drivers
- Attempting to load and execute boot selections

Rust std implementation can be used here.

## TSL – Transient System Load
This is the stage between boot device selection and handoff to the OS. At this point, one may enter the UEFI shell or execute a UEFI application such as the OS boot loader.

Rust std implementation can be used here.

## RT – Runtime
The UEFI hands off to the operating system (OS) after `ExitBootServices()` is executed. A UEFI-compatible OS is now responsible for exiting boot services triggering the firmware to unload all no longer needed code and data, leaving only runtime services code/data, e.g., SMM and ACPI.[86] A typical modern OS will prefer to use its own programs (such as kernel drivers) to control hardware devices.

Since any UEFI stuff still running needs to be started in one of the earlier stages, it is possible to use Rust std. However, it is up to the developer to deal with dangling/invalid pointers once `ExitBootServices` is called.

Also, the allocator, console, and a lot of other stuff are unavailable at this point. While the std will try to ensure the invalid pointers are not accessed, it cannot provide functionality that no longer exists. 

I will not recommend using Rust std for writing Runtime Drivers unless you are absolutely sure of what you are doing.

## AL - After Life
The After Life (AL) phase consists of persistent UEFI drivers used for storing the state of the system during the OS orderly shutdown, sleep, hibernate or restart processes.

Similar to RT, I would not recommend using Rust std in this phase.

<br>

# Handling ExitBootServices()
The Rust std now registers a callback when `EVT_SIGNAL_EXIT_BOOT_SERVICES` is signaled. This callback makes BootServices unavailable. The most significant consequence of this is that the provided [Allocator](https://doc.rust-lang.org/stable/std/alloc/trait.GlobalAlloc.html) no longer works.

Since, all the memory in Rust std is allocated with the memory type `EfiLoaderData`, it is not really valid at after this point. Feel free to comment at the [PR](https://github.com/rust-lang/rust/pull/100316) if you would like to see this configurable.

An additional function, `boot_services` has been added under `std::os::uefi::env`. This function ensures that BootServices are still valid before returning the pointer. However, it is still possible to circumvent this by just directly using the `system_table` function.

# Conclusion
Hopefully, this post will clarify when and where Rust std for UEFI should be used.

Consider [supporting me](@/pages/supportme.md) if you like my work.

# Helpful Resources
- [Boot Sequence](https://edk2-docs.gitbook.io/edk-ii-build-specification/2_design_discussion/23_boot_sequence)
- [Minimal Rust std PR](https://github.com/rust-lang/rust/pull/100316)
