+++
title = "Using Rust main from a custom entry point (Part 2)"
description = "How to use Rust main function with a custom entry point with a custom function signature"
date = "2022-06-25T22:28:12+05:30"

[taxonomies]
categories = ["post"]
tags = ["rust", "tianocore", "gsoc22", "uefi"]
+++

Hello everyone. I will continue where I left off in [post](@/post7.md) with the Rust main in this post. Since we now have an allocator, the [`Thread::new`](https://github.com/rust-lang/rust/blob/00ce47209dfdd8ef8871c6ec804f0e0e04d10702/library/std/src/rt.rs#L85) statement at `library/std/src/rt.rs` works. So we need to fix the line where we set the main thread with guard information. This will be a short post since it turned out easier than I initially thought.

<!-- more -->

# Thread Local Macro
The macro [`thread_local!`](https://doc.rust-lang.org/std/macro.thread_local.html) wraps any number of static declarations and makes them thread-local. To get a better understanding of what this macro does, we can take a look at this example from the docs:
```rust
use std::cell::RefCell;
use std::thread;

thread_local!(static FOO: RefCell<u32> = RefCell::new(1));

FOO.with(|f| {
    assert_eq!(*f.borrow(), 1);
    *f.borrow_mut() = 2;
});

// each thread starts out with the initial value of 1
let t = thread::spawn(move|| {
    FOO.with(|f| {
        assert_eq!(*f.borrow(), 1);
        *f.borrow_mut() = 3;
    });
});

// wait for the thread to complete and bail out on panic
t.join().unwrap();

// we retain our original value of 2 despite the child thread
FOO.with(|f| {
    assert_eq!(*f.borrow(), 2);
});
```
As you can see, it allows the creation of static variables local to each thread.

<br>

# The Problem
The `std::sys_common::thread_info` module defines a thread-local variable `THREAD_INFO` using the `thread_local!` macro. The error occurs when this variable is lazily initialized in the `init()` function at `rt.rs`.

<br>

# The Solution
The `__thread_local_internal` macro contains conditional compilation for `wasm` target without `atomics`. Since UEFI is single-threaded (I know there is a way to execute code in other cores, but it's not exactly true multi-threading, from what I understand), and I don't have any `atomics` (at least not yet), I just decided to use the wasm implementation. This maps to a simple mutable static which should be fine to do for now. The new `__thread_local_internal` looks like:
```rust
#[doc(hidden)]
#[unstable(feature = "thread_local_internals", reason = "should not be necessary", issue = "none")]
#[macro_export]
#[allow_internal_unstable(thread_local_internals, cfg_target_thread_local, thread_local)]
#[allow_internal_unsafe]
macro_rules! __thread_local_inner {
    // used to generate the `LocalKey` value for const-initialized thread locals
    (@key $t:ty, const $init:expr) => {{
        #[cfg_attr(not(windows), inline)] // see comments below
        #[deny(unsafe_op_in_unsafe_fn)]
        unsafe fn __getit(
            _init: $crate::option::Option<&mut $crate::option::Option<$t>>,
        ) -> $crate::option::Option<&'static $t> {
            const INIT_EXPR: $t = $init;

            // uefi and wasm without atomics maps directly to `static mut`, and dtors
            // aren't implemented because thread dtors aren't really a thing
            // on wasm right now
            //
            // FIXME(#84224) this should come after the `target_thread_local`
            // block.
            #[cfg(any(all(target_family = "wasm", not(target_feature = "atomics")), target_os = "uefi"))]
            {
                static mut VAL: $t = INIT_EXPR;
                unsafe { $crate::option::Option::Some(&VAL) }
            }

            // If the platform has support for `#[thread_local]`, use it.
            #[cfg(all(
                target_thread_local,
                not(all(target_family = "wasm", not(target_feature = "atomics"))),
                not(target_os = "uefi")
            ))]
            {
                #[thread_local]
                static mut VAL: $t = INIT_EXPR;

                // If a dtor isn't needed we can do something "very raw" and
                // just get going.
                if !$crate::mem::needs_drop::<$t>() {
                    unsafe {
                        return $crate::option::Option::Some(&VAL)
                    }
                }

                // 0 == dtor not registered
                // 1 == dtor registered, dtor not run
                // 2 == dtor registered and is running or has run
                #[thread_local]
                static mut STATE: $crate::primitive::u8 = 0;

                unsafe extern "C" fn destroy(ptr: *mut $crate::primitive::u8) {
                    let ptr = ptr as *mut $t;

                    unsafe {
                        $crate::debug_assert_eq!(STATE, 1);
                        STATE = 2;
                        $crate::ptr::drop_in_place(ptr);
                    }
                }

                unsafe {
                    match STATE {
                        // 0 == we haven't registered a destructor, so do
                        //   so now.
                        0 => {
                            $crate::thread::__FastLocalKeyInner::<$t>::register_dtor(
                                $crate::ptr::addr_of_mut!(VAL) as *mut $crate::primitive::u8,
                                destroy,
                            );
                            STATE = 1;
                            $crate::option::Option::Some(&VAL)
                        }
                        // 1 == the destructor is registered and the value
                        //   is valid, so return the pointer.
                        1 => $crate::option::Option::Some(&VAL),
                        // otherwise the destructor has already run, so we
                        // can't give access.
                        _ => $crate::option::Option::None,
                    }
                }
            }

            // On platforms without `#[thread_local]` we fall back to the
            // same implementation as below for os thread locals.
            #[cfg(all(
                not(target_thread_local),
                not(all(target_family = "wasm", not(target_feature = "atomics"))),
                not(target_os = "uefi")
            ))]
            {
                #[inline]
                const fn __init() -> $t { INIT_EXPR }
                static __KEY: $crate::thread::__OsLocalKeyInner<$t> =
                    $crate::thread::__OsLocalKeyInner::new();
                #[allow(unused_unsafe)]
                unsafe {
                    __KEY.get(move || {
                        if let $crate::option::Option::Some(init) = _init {
                            if let $crate::option::Option::Some(value) = init.take() {
                                return value;
                            } else if $crate::cfg!(debug_assertions) {
                                $crate::unreachable!("missing initial value");
                            }
                        }
                        __init()
                    })
                }
            }
        }

        unsafe {
            $crate::thread::LocalKey::new(__getit)
        }
    }};

    // used to generate the `LocalKey` value for `thread_local!`
    (@key $t:ty, $init:expr) => {
        {
            #[inline]
            fn __init() -> $t { $init }

            // When reading this function you might ask "why is this inlined
            // everywhere other than Windows?", and that's a very reasonable
            // question to ask. The short story is that it segfaults rustc if
            // this function is inlined. The longer story is that Windows looks
            // to not support `extern` references to thread locals across DLL
            // boundaries. This appears to at least not be supported in the ABI
            // that LLVM implements.
            //
            // Because of this we never inline on Windows, but we do inline on
            // other platforms (where external references to thread locals
            // across DLLs are supported). A better fix for this would be to
            // inline this function on Windows, but only for "statically linked"
            // components. For example if two separately compiled rlibs end up
            // getting linked into a DLL then it's fine to inline this function
            // across that boundary. It's only not fine to inline this function
            // across a DLL boundary. Unfortunately rustc doesn't currently
            // have this sort of logic available in an attribute, and it's not
            // clear that rustc is even equipped to answer this (it's more of a
            // Cargo question kinda). This means that, unfortunately, Windows
            // gets the pessimistic path for now where it's never inlined.
            //
            // The issue of "should enable on Windows sometimes" is #84933
            #[cfg_attr(not(windows), inline)]
            unsafe fn __getit(
                init: $crate::option::Option<&mut $crate::option::Option<$t>>,
            ) -> $crate::option::Option<&'static $t> {
                #[cfg(any(all(target_family = "wasm", not(target_feature = "atomics")), target_os = "uefi"))]
                static __KEY: $crate::thread::__StaticLocalKeyInner<$t> =
                    $crate::thread::__StaticLocalKeyInner::new();

                #[thread_local]
                #[cfg(all(
                    target_thread_local,
                    not(all(target_family = "wasm", not(target_feature = "atomics"))),
                    not(target_os = "uefi")
                ))]
                static __KEY: $crate::thread::__FastLocalKeyInner<$t> =
                    $crate::thread::__FastLocalKeyInner::new();

                #[cfg(all(
                    not(target_thread_local),
                    not(all(target_family = "wasm", not(target_feature = "atomics"))),
                    not(target_os = "uefi")
                ))]
                static __KEY: $crate::thread::__OsLocalKeyInner<$t> =
                    $crate::thread::__OsLocalKeyInner::new();

                // FIXME: remove the #[allow(...)] marker when macros don't
                // raise warning for missing/extraneous unsafe blocks anymore.
                // See https://github.com/rust-lang/rust/issues/74838.
                #[allow(unused_unsafe)]
                unsafe {
                    __KEY.get(move || {
                        if let $crate::option::Option::Some(init) = init {
                            if let $crate::option::Option::Some(value) = init.take() {
                                return value;
                            } else if $crate::cfg!(debug_assertions) {
                                $crate::unreachable!("missing default value");
                            }
                        }
                        __init()
                    })
                }
            }

            unsafe {
                $crate::thread::LocalKey::new(__getit)
            }
        }
    };
    ($(#[$attr:meta])* $vis:vis $name:ident, $t:ty, $($init:tt)*) => {
        $(#[$attr])* $vis const $name: $crate::thread::LocalKey<$t> =
            $crate::__thread_local_inner!(@key $t, $($init)*);
    }
}
```

We will also need to add `target_os = "uefi"` to conditional compilation of `std::thread::__StaticLocalKeyInner` and `std::thread::local::statik`.

After that, it works perfectly. I'm not sure if this is the correct implementation, but it also fixes stdio (which I will implement in the next post) for me, so I think it is acceptable for now. However, anyone who understands this better is free to contact me through mail.

<br>

# Conclusion
As I stated earlier, this is a pretty short post. While I could post an empty `main` function, it's useless without having the ability to print to screen from `main()`. So this is where I will conclude for now. I promise we will print "Hello World!" from `main()` next time.
