+++
title = "Using Rust main from custom entry point (Part 3)"
description = "Generating a compiler generated entry point"
date = "2022-10-27T22:22:12+05:30"
update_date = "2022-09-06"

[taxonomies]
tags = ["rust", "uefi"]
+++
Even though GSoC 2022 has ended, I am still working on getting [Rust std PR](https://github.com/rust-lang/rust/pull/100316) merged into the master. In the initial PR, I still used the entry function described in the [prior post](@/blog/post7.md). However, this had many limitations, and thus I have finally transitioned to using a compiler-generated entry function.

<!-- more -->

# Limitations of the older method
In the initial PR, I used an extern `efi_main` function called the compiler-generated `main` function. This had the following limitations:
1. The `efi_main` function is generated in all projects using std. This means it will be present when using the `no_main` attribute or library crate, thus diminishing the usefulness of std in libraries and drivers. It also rules out anything that does not want to use Heap allocation since Rust does some Heap allocations during the runtime setup.
2. Trying to pass pointers to `SystemTable` and `ImageHandle` as argv caused errors when building in release mode. This meant I was doing initialization from a function with no runtime setup and could thus cause all kinds of UB in case of panics.
3. Added a layer of redirection.

# The new implementation
If you are unaware of everything that goes on before Rust `main`, then you should read my [previous post](@/blog/post7.md) about this. I am replacing the compiler-generated C `main` in the new implementation with the Win64 `efi_main` function. Inspecting the LLVM-IR of a simple application, we can see the generated `efi_main`:
```asm
; Function Attrs: noredzone nounwind
define win64cc i64 @efi_main(ptr %0, ptr %1) unnamed_addr #4 {
top:
  %_9.i = alloca ptr, align 8
  %2 = alloca [2 x ptr], align 8
  store ptr %0, ptr %2, align 8
  %3 = getelementptr inbounds ptr, ptr %2, i64 1
  store ptr %1, ptr %3, align 8
  call void @llvm.lifetime.start.p0(i64 8, ptr nonnull %_9.i)
  store ptr @_ZN11hello_world4main17h57c4996130ab67deE, ptr %_9.i, align 8
; call std::rt::lang_start_internal
  %4 = call i64 @_ZN3std2rt19lang_start_internal17hd4c288149f777b4aE(ptr noundef nonnull align 1 %_9.i, ptr noalias noundef nonnull readonly align 8 dereferenceable(24) @vtable.0, i64 2, ptr nonnull %2, i8 2) #6
  call void @llvm.lifetime.end.p0(i64 8, ptr nonnull %_9.i)
  ret i64 %4
}
```
What this function does is pass an array `[ImageHandle, *mut SystemTable]` as `argv` and `2` as `argc` to Rust `lang_start`. We then initialize the global variables from Rust `sys::init()` function.

# Benefits of new implementation
1. `no_main` attribute works as intended now since `std::os::uefi::init()` is part of public API, this allows using Rust std in cases where we want to initialize something before the Heap is available (in some driver) but would still like to use `std` after initialization.
2. Allows UEFI to work like all other Rust platforms since `efi_main` is only generated in places where C `main` would typically be generated.
3. It seems to improve startup speed.

# Conclusion
With this implementation, one of the major blockers for the PR has been resolved. While implementing this, I also modified the Rust target specification to allow specifying the Entry Function name and ABI, which might enable making compiler-generated entry functions easier for other targets. Feel free to comment on the PR and follow it if interested.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful Links
1. [Rust std PR](https://github.com/rust-lang/rust/pull/100316) 
2. [Using Rust main from a custom entry point](@/blog/post7.md)
