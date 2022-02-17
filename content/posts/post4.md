+++
title = "Using KConfig with Rust"
description = "A guide on how to use KConfig KDE Framework from Rust"
date = "2022-02-12T01:54:01+05:30"
draft = true

[taxonomies]
tags = ["rust", "kde", "sok22"]
+++

# Background

Hello everyone, I am currently working on KConfig bindings for Rust as a part of Season of KDE 2022. The wrappers for most of the major parts of KConfig are complete, so I decided to rewrite the [Introduction to KConfig Docs](https://develop.kde.org/docs/use/configuration/introduction/) in Rust. The bindings are still not stable and will probably change before the end of Season of KDE. Still, this post should also help me test out the bindings outside tests. The kconfig bindings can be found [here](https://invent.kde.org/oreki/kconfig-rs).

<!-- more -->

Currently, the bindings use the git version of [qttypes](https://github.com/woboq/qmetaobject-rs) since I had to merge some changes upstream that are needed for these bindings. So they are not ready for prime time just yet. But I should be able to switch to crates.io version as soon as a new version of qttypes is published.

# The KConfig Class

The KConfig object is used to access a given configuration object. The config object can be created in the following ways:

```rust
KConfig::new("data", kconfig::OpenFlags::SimpleConfig, qttypes::QStandardPathLocation::AppDataLocation);
```
