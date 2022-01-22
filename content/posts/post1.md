+++
title = "Creating Rust/QML Project"
description = "Using cargo-generate to create Rust/QML Project."
date = "2022-01-22"

[taxonomies]
tags = ["rust", "kde"]
+++
# Background
For the last few months, I have been pushing Rust/QT development along. I am the author of [ki18n](https://github.com/Ayush1325/ki18n-rs) crate and am currently in the middle of creating [kconfig](https://invent.kde.org/oreki/kconfig-rs) crate as a part of [Season of KDE 2022](https://season.kde.org/).

In this post, I will walk you through creating a new Rust/QML project using [cargo-generate](https://github.com/cargo-generate/cargo-generate) templates. I made these templates to encourage more people to test out Qt development with Rust.

<!-- more -->

# Install Cargo Generate
Full Instructions are present in the repository [here](https://github.com/cargo-generate/cargo-generate#installation).

## Using Cargo
### With System's OpenSSL
```sh
cargo install cargo-generate
```
### With vendored OpenSSL
```sh
cargo install cargo-generate --features vendored-openssl
```
### Arch Linux
```sh
pacman -S cargo-generate
```
### Manual
1. Download the binary tarball for your platform from [releases page](https://github.com/cargo-generate/cargo-generate/releases).
2. Unpack the tarball and place the binary `cargo-generate` in `~/.cargo/bin/`

# Select the Template
Currently, there are two templates I have created for Rust/QML. The templates can be found [here](https://invent.kde.org/oreki/rust-qt-template).

## Basic QtQuick Application
This template creates a basic QtQuick Application with Rust. It does not contain any KDE Components. This template should work in all platforms QT, and Rust is supported.

### Dependencies
#### Manual
QT can be installed using the [installer](https://www.qt.io/download). Be sure to install `qmake` since it is used by [qmetaobject](https://crates.io/crates/qmetaobject) crate to detect system QT install.
#### Ubuntu
```sh
sudo apt install build-essential qtbase5-dev qtdeclarative5-dev libqt5svg5-dev qtquickcontrols2-5-dev qml-module-qtquick-layouts
```

### Generate Project
```sh
cargo generate --git https://invent.kde.org/oreki/rust-qt-template/ basic-qtquick --name myproject
```

### Run Project
We will also set `RUST_LOG` variable for enabling logs.
```sh
RUST_LOG=error,warn,info,debug,trace cargo run
```

### Screenshots
{{ image(src="/images/post1/basic-qt1.png")}}

## Kirigami Application
This template creates a basic Kirigami Application with Rust. Currently, this template is only tested in Linux. It can technically work in other platforms if the KDE Frameworks path is manually specified, but I have not tried that. If you would like to help, the crate for detecting KDE Frameworks is [kde_frameworks](https://crates.io/crates/kde_frameworks).

### Dependencies
#### Manual
1. `QT` can be installed using the [installer](https://www.qt.io/download). Be sure to install `qmake` since it is used by [qmetaobject](https://crates.io/crates/qmetaobject) crate to detect system `QT` install. 
2. KDE Frameworks (`Kirigami` and `KI18n`) should also be installed. Be sure to install `kf5-config` since it is used to locate `Kirigami` and `KI18n` Frameworks.
#### Ubuntu
```sh
sudo apt install build-essential qtbase5-dev qtdeclarative5-dev libqt5svg5-dev qtquickcontrols2-5-dev qml-module-qtquick-layouts qml-module-org-kde-kirigami2 kirigami2-dev libkf5i18n-dev gettext libkf5coreaddons-dev libkf5kdelibs4support5-bin
```
### Arch-based
```sh
sudo pacman -Syu base-devel extra-cmake-modules cmake kirigami2 kde-sdk-meta gettext
```
### Fedora
```sh
sudo dnf groupinstall "Development Tools" "Development Libraries"
sudo dnf install extra-cmake-modules cmake qt5-qtbase-devel qt5-qtdeclarative-devel qt5-qtquickcontrols2-devel kf5-kirigami2 kf5-kirigami2-devel kf5-ki18n-devel kf5-kcoreaddons-devel gettext
```

### Generate Project
```sh
cargo generate --git https://invent.kde.org/oreki/rust-qt-template/ kirigami --name myproject
```

### Run Project
We will also set `RUST_LOG` variable for enabling logs.
```sh
RUST_LOG=error,warn,info,debug,trace cargo run
```

### Screenshots
{{ image(src="/images/post1/kirigami1.png")}}

# Conclusion
If you find this exciting or want to try something new in Rust/QT, here is a list of crates related to Rust + QT development.
1. [qmetaobject](https://crates.io/crates/qmetaobject)
2. [Rust Qt Binding Generattor](https://invent.kde.org/sdk/rust-qt-binding-generator)
3. [ki18n](https://github.com/Ayush1325/ki18n-rs)
4. [kconfig](https://invent.kde.org/oreki/kconfig-rs)
5. [rust-qt-template](https://invent.kde.org/oreki/rust-qt-template)
