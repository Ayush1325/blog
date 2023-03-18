+++
title = "Compiler generate custom entry for any target"
description = "Guide to use custom entry function for any target"
date = "2023-03-18T16:08:12+05:30"

[taxonomies]
categories = ["post"]
tags = ["rust"]
+++
Hello everyone. While implementing Rust std for UEFI, I came across an interesting problem. The signature of the entry function in Rust did not match the UEFI entry signature. Initially, I just used a hacky way to get things to work. However, later I made some changes to upstream Rust to make generating entry functions with custom signatures much easier. In this post, I will show how to implement generating such an entry function in upstream Rust through the example of UEFI.

<!-- more -->

Read my [prior post](@/post7.md) for an in-depth explanation of everything that goes on before Rust main.

# Background
UEFI expects the entry function to have the following signature:
```rust
use r_efi::efi;

extern "efiapi" fn _start(image_handle: efi::Handle, system_table: *mut efi::SystemTable) -> efi::Status;
```
Currently, the Rust compiler generates a custom LLVM function that calls the Rust entry. In earlier versions of Rust, generating a custom entry function would have needed modification to many different modules in upstream Rust. However, since [#104045](https://github.com/rust-lang/rust/pull/104045) and [#104001](https://github.com/rust-lang/rust/pull/104001) have been merged, only a few files need to be modified.

It is important to note that the Rust `lang_start` has the following signature:
```rust
fn lang_start<T: crate::process::Termination + 'static>(
    main: fn() -> T,
    argc: isize,
    argv: *const *const u8,
) -> isize;
```
Thus even a custom entry function has to finally call this `lang_start`.

# Implementation
Now I will go through the two main steps needed for custom entry function.

## Modifying Target spec
The target spec now contains the entry function's **name** and **ABI**. This means it is also possible to modify the name and ABI by using a target specification Json instead of modifying the compiler.

For the UEFI entry name, we need to modify `compiler/rustc_target/src/spec/uefi_msvc_base.rs`:
```patch
diff --git a/compiler/rustc_target/src/spec/uefi_msvc_base.rs b/compiler/rustc_target/src/spec/uefi_msvc_base.rs
index 8968d3c8fc1..a50a55ad7e0 100644
--- a/compiler/rustc_target/src/spec/uefi_msvc_base.rs
+++ b/compiler/rustc_target/src/spec/uefi_msvc_base.rs
@@ -46,6 +46,7 @@ pub fn opts() -> TargetOptions {
         stack_probes: StackProbeType::Call,
         singlethread: true,
         linker: Some("rust-lld".into()),
+        entry_name: "efi_main".into(),
         ..base
     }
 }
```

For the entry ABI, we need to modify `compiler/rustc_target/src/spec/x86_64_unknown_uefi.rs`:
```patch
diff --git a/compiler/rustc_target/src/spec/x86_64_unknown_uefi.rs b/compiler/rustc_target/src/spec/x86_64_unknown_uefi.rs
index a7ae17839da..9bc74ad2999 100644
--- a/compiler/rustc_target/src/spec/x86_64_unknown_uefi.rs
+++ b/compiler/rustc_target/src/spec/x86_64_unknown_uefi.rs
@@ -5,12 +5,13 @@
 // The win64 ABI is used. It differs from the sysv64 ABI, so we must use a windows target with
 // LLVM. "x86_64-unknown-windows" is used to get the minimal subset of windows-specific features.

-use crate::spec::Target;
+use crate::{abi::call::Conv, spec::Target};

 pub fn target() -> Target {
     let mut base = super::uefi_msvc_base::opts();
     base.cpu = "x86-64".into();
     base.max_atomic_width = Some(64);
+    base.entry_abi = Conv::X86_64Win64;

     // We disable MMX and SSE for now, even though UEFI allows using them. Problem is, you have to
     // enable these CPU features explicitly before their first use, otherwise their instructions
```

## Implement LLVM entry generation
Now we need to implement the actual LLVM code to generate for our target. For UEFI, we will pass the SystemTable and ImageHandle in `argv` and set the value of `argc` to 2. For this, we need to make the following changes to `compiler/rustc_codegen_ssa/src/base.rs`:
```patch
diff --git a/compiler/rustc_codegen_ssa/src/base.rs b/compiler/rustc_codegen_ssa/src/base.rs
index abc510e360d..8be75552efd 100644
--- a/compiler/rustc_codegen_ssa/src/base.rs
+++ b/compiler/rustc_codegen_ssa/src/base.rs
@@ -430,9 +430,11 @@ fn create_entry_fn<'a, 'tcx, Bx: BuilderMethods<'a, 'tcx>>(
         rust_main_def_id: DefId,
         entry_type: EntryFnType,
     ) -> Bx::Function {
-        // The entry function is either `int main(void)` or `int main(int argc, char **argv)`,
-        // depending on whether the target needs `argc` and `argv` to be passed in.
-        let llfty = if cx.sess().target.main_needs_argc_argv {
+        // The entry function is either `int main(void)` or `int main(int argc, char **argv)`, or
+        // `Status efi_main(Handle hd, SystemTable *st)` depending on the target.
+        let llfty = if cx.sess().target.os.contains("uefi") {
+            cx.type_func(&[cx.type_i8p(), cx.type_i8p()], cx.type_isize())
+        } else if cx.sess().target.main_needs_argc_argv {
             cx.type_func(&[cx.type_int(), cx.type_ptr_to(cx.type_i8p())], cx.type_int())
         } else {
             cx.type_func(&[], cx.type_int())
@@ -496,8 +498,12 @@ fn create_entry_fn<'a, 'tcx, Bx: BuilderMethods<'a, 'tcx>>(
         };

         let result = bx.call(start_ty, None, start_fn, &args, None);
-        let cast = bx.intcast(result, cx.type_int(), true);
-        bx.ret(cast);
+        if cx.sess().target.os.contains("uefi") {
+            bx.ret(result);
+        } else {
+            let cast = bx.intcast(result, cx.type_int(), true);
+            bx.ret(cast);
+        }

         llfn
     }
@@ -508,7 +514,18 @@ fn get_argc_argv<'a, 'tcx, Bx: BuilderMethods<'a, 'tcx>>(
     cx: &'a Bx::CodegenCx,
     bx: &mut Bx,
 ) -> (Bx::Value, Bx::Value) {
-    if cx.sess().target.main_needs_argc_argv {
+    if cx.sess().target.os.contains("uefi") {
+        // Params for UEFI
+        let param_handle = bx.get_param(0);
+        let param_system_table = bx.get_param(1);
+        let arg_argc = bx.const_int(cx.type_isize(), 2);
+        let arg_argv = bx.alloca(cx.type_array(cx.type_i8p(), 2), Align::ONE);
+        bx.store(param_handle, arg_argv, Align::ONE);
+        let arg_argv_el1 =
+            bx.gep(cx.type_ptr_to(cx.type_i8()), arg_argv, &[bx.const_int(cx.type_int(), 1)]);
+        bx.store(param_system_table, arg_argv_el1, Align::ONE);
+        (arg_argc, arg_argv)
+    } else if cx.sess().target.main_needs_argc_argv {
         // Params from native `main()` used as args for rust start function
         let param_argc = bx.get_param(0);
         let param_argv = bx.get_param(1);
```

**Note:** While the generation code seems to use `i8` pointers, the generated LLVM IR actually uses [Opque Pointers](https://llvm.org/docs/OpaquePointers.html).

# Conclusion
As outlined in this post, it should now be much easier to implement generating custom entry functions for Rust targets. This can be especially useful for embedded or some custom OS use cases.

Consider [supporting me](@/pages/supportme.md) if you like my work.
