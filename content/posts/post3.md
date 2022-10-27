+++
title = "Bitflags in Rust"
description = "Representing QFlags in Rust"
date = "2022-01-30T01:54:01+05:30"

[taxonomies]
tags = ["rust", "kde", "cpp", "sok22"]
+++

While working on Rust bindings for KConfig as a part of Season of KDE 2022, I came across a few problems while trying to represent [`QFlags`](https://doc.qt.io/qt-5/qflags.html) in Rust:

1. Most `QFlags` are defined as C++ enums in which multiple members can have the same value. This is not possible in Rust enum.
2. It is possible to enable multiple flags using BitwiseOr. Rust enums cannot do bitwise operations.

<!-- more -->

This post will guide you through the various implementations I came up with and their tradeoffs.

# The C++ enum

The enum I was trying to implement was [`KConfig::OpenFlags`](https://api.kde.org/frameworks/kconfig/html/classKConfig.html#ad1f23964bbf8c11449e92a2596d15f7e). The enum is given below:

```cpp
enum OpenFlag {
    IncludeGlobals = 0x01, ///< Blend kdeglobals into the config object.
    CascadeConfig = 0x02, ///< Cascade to system-wide config files.
    SimpleConfig = 0x00, ///< Just a single config file.
    NoCascade = IncludeGlobals, ///< Include user's globals, but omit system settings.
    NoGlobals = CascadeConfig, ///< Cascade to system settings, but omit user's globals.
    FullConfig = IncludeGlobals | CascadeConfig, ///< Fully-fledged config, including globals and cascading to system settings
};
```

# Implementation 1: Using Rust modules

This method uses a combination of Rust modules and consants. The sample implementation is as follow:

```rust
pub mod OpenFlags {
    type E = u32;
    const INCLUDE_GLOBALS: Self::E = 0x01;
    const CASCADE_CONFIG: Self::E = 0x02;
    const SIMPLE_CONFIG: Self::E = 0x00;
    const NO_CASCASE: Self::E = Self::INCLUDE_GLOBALS;
    const NO_GLOBALS: Self::E = Self::CASCADE_CONFIG;
    const FULL_CONFIG: Self::E = Self::INCLUDE_GLOBALS | Self::CASCADE_CONFIG;
}

fn something(flag: OpenFlags::E) {}
```

## Advantages

1. Const is replaced at compile time, so no performance cost.

2. All values can be documented in the same way using Rust comments.

3. Multiple flags can be activated.

## Drawbacks

1. Not an enum. Just a collection of constants.

# Implementation 2: Using const in Impl

This method defines the problematic members as `const` in `impl`. The sample implementation is as follows:

```rust
#[repr(C)]
pub enum OpenFlags {
    IncludeGlobals = 0x01,
    CascadeConfig = 0x02,
    SimpleConfig = 0x00,
    FullConfig = 0x01 | 0x02,
}

#[allow(non_upper_case_globals)]
impl OpenFlags {
    const NoCascade: Self = Self::IncludeGlobals;
    const NoGlobals: Self = Self::CascadeConfig;
}

fn something(flag: OpenFlags) {}
```

## Advantages

1. Enum, for the most part.

## Drawbacks

1. Inconsistent documentation. The constants don't show up as enum variants.
2. Multiple flags cannot be activated

# Implementation 3: Converting standard Rust enums when passing to C++

This method uses standard rust enums. The sample implementation is as follows:

```rust
pub enum OpenFlags {
    IncludeGlobals,
    CascadeConfig,
    SimpleConfig,
    NoCascade,
    NoGlobals,
    FullConfig
}

impl OpenFlags {
    type E = u32;
    const INCLUDE_GLOBALS: Self::E = 0x01;
    const CASCADE_CONFIG: Self::E = 0x02;
    const SIMPLE_CONFIG: Self::E = 0x00;

    pub fn to_cpp(&self) -> Self::E {
        match self {
            Self::IncludeGlobals => Self::INCLUDE_GLOBALS,
            Self::CascadeConfig => Self::CASCADE_CONFIG,
            Self::SimpleConfig => Self::SIMPLE_CONFIG,
            Self::NoCascade => Self::INCLUDE_GLOBALS,
            Self::NoGlobals => Self::CASCADE_CONFIG,
            Self::FullConfig => Self::INCLUDE_GLOBALS | Self::CASCADE_CONFIG,
        }
    }
}

fn something(flag: OpenFlags) {
    let flag = flag.to_cpp();
    ...
}
```

## Advantages

1. Completely Enum.

2. Documentation works as expected.

## Drawbacks

1. Function call every time passing from Rust to C++. I don't think this will have much performance penalty, but still worth mentioning.

2. Cannot set multiple flags at once. Eg `OpenFlag::IncludeGlobal | OpenFlag::CascadeConfig` not possible

# Implementation 4: use [bitflags](https://crates.io/crates/bitflags) crate

This is the implementation that I finally settled on. The implementation is as follows:

```rust
use bitflags::bitflags

bitflags! {
    /// Determines how the system-wide and user's global settings will affect the reading of the configuration.
    /// This is a bitfag. Thus it is possible to pass options like `OpenFlags::INCLUDE_GLOBALS |
    /// OpenFlags::CASCADE_CONFIG`
    #[repr(C)]
    pub struct OpenFlags: u32 {
        /// Blend kdeglobals into the config object.
        const INCLUDE_GLOBALS = 0x01;
        /// Cascade to system-wide config files.
        const CASCADE_CONFIG = 0x02;
        /// Just a single config file.
        const SIMPLE_CONFIG = 0x00;
        /// Include user's globals, but omit system settings.
        const NO_CASCADE = Self::INCLUDE_GLOBALS.bits;
        /// Cascade to system settings, but omit user's globals.
        const NO_GLOBALS = Self::CASCADE_CONFIG.bits;
        /// Fully-fledged config, including globals and cascading to system settings.
        const FULL_CONFIG = Self::INCLUDE_GLOBALS.bits | Self::CASCADE_CONFIG.bits;
    }
}

fn something(flag: OpenFlags) {}
```

## Advantages

1. Multiple flags can be used together.

2. Documentation is consistent.

## Drawbacks

1. Not enum. Shows up as `struct` in docs.

# Documentation Screenshot

{{ image(src="/images/post2/docs.webp") }}

# Conclusion

I think I will be using [bitflags](https://crates.io/crates/bitflags) for representing all `QFlags` in [kconfig](https://invent.kde.org/oreki/kconfig-rs/-/tree/master) for the foreseeable future.
