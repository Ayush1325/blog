+++
title = "Season of KDE 2022"
description = "My Journey with KDE"
date = "2022-01-26T20:00:33+05:30"

[taxonomies]
tags = ["kde", "sok22"]
+++

I am Ayush Singh, a second-year student of the Indian Institute of Technology, Dhanbad, India. My application has been accepted in the Season of KDE 2022. I will be working on writing a Rust wrapper for KConfig KDE Framework. This post describes my journey with KDE and why I submitted this Project for the Season of KDE.

<!-- more -->

# My Introduction to Linux

I was introduced to the world of Linux back in 2016 when I was working on compiling android from source. Like most people, I started with Ubuntu. I didn't explore Linux much at the time other than building android, and that was it.

However, things changed in 2018 when I got my PC. Since I already knew about Linux, I decided to dual-boot and explore a new operating system. Slowly, I stopped using the Windows partition and removed it altogether in early 2020.

# My Introduction to KDE

I was first introduced to KDE when I switched to OpenSUSE Tumbleweed. I didn't pay much attention to KDE at the time and quickly moved on. After this, I started using standalone window managers (like i3, AwesomeWM) for quite a while. The next time I actively used KDE was in the summer of 2020 when the lockdown began, and my school exams got postponed indefinitely. During that time, I discovered KDE Activities which completely changed my workflow. Since then, I have tried Gnome and standalone windows managers; however, none seem to offer the same level of coherence and flexibility as KDE. Also, did I already mention how much I love KDE Activities?

As for KDE Development, I am somewhat inexperienced. I have also opened Bugs in KDE and two merge requests, but they were pretty minor. Season of KDE will also allow me to get better acquainted with KDE Development.

# Past Experience with Qt

I have used Qt Framework to write GUI applications in C++ for my School projects.

1. In 2019, I made a [Calculator](https://github.com/Ayush1325/Calculator).
2. In 2020, I made a [Library Management](https://github.com/Ayush1325/library_management) software.

Both of them used Qt Widgets since I didn't know about QML at the time.

# My Introduction to Rust

I had been interested in Systems Programming in the past. During the lockdown, I came across Rust Language. I quickly got interested in its promise of performance and safety and being more modern than C in general. After reading the [Rust Book](https://doc.rust-lang.org/book/), I used Rust to create simple projects, like a web crawler and a magic bytes reader. I now understand that, like all programming languages, Rust has its strengths and weaknesses. However, I still love working with Rust.

# Trying to use Rust in KDE

In June of 2021, I thought about writing a Web Browser for some reason. I did not want to use the Blink engine. I had previously heard about [Servo](https://github.com/servo/servo) and decided to use it. While doing that, I started searching for GUI toolkits in Rust since Servo is written in Rust. I wanted to use Qt with Rust, but I quickly found out that using Rust with Qt was more difficult than I would like. At around the same time, Gtk-rs was made official with Gtk4, which seemed to work great. I tried switching to Gnome for a while after that but ultimately failed.

Sadly, I never created that web browser. However, I did come out with a new conviction. I decided to make the missing tooling required to develop KDE/QML applications with Rust without writing any C++.

I started by looking into already existing crates for Qt development with Rust. I found [qmetaobject](https://github.com/woboq/qmetaobject-rs) crate, which was closely aligned with how I wanted Rust + Qt development to be. To create Kirigami applications using qmetaobject, I created bindings for the KI18n Localization framework ([kconfig crate](https://github.com/Ayush1325/ki18n-rs)). It was used with almost all Kirigami applications and thus was a natural starting point. I also created a Rust crate for linking Kirigami Framework to the Rust application. While working on these bindings, I also contributed the qmetaobject upstream since some of the methods and enums I required were missing for qmetaobject. This helped me get familiar with Rust memory management, which I had previously taken for granted.

# Applying for Season of KDE

I already knew about the Season of KDE and decided to apply for it. I submitted a proposal about writing "Rust wrapper of KConfig". Since I was already working on the KI18n wrapper, I had the necessary experience to make this possible.

After some searching in the KDE mailing list, I came across Jos van den Oever, the author of [Rust Qt Bindings Generator](https://invent.kde.org/sdk/rust-qt-binding-generator). He agreed to become my mentor for the Season of KDE. This project might have been delayed if he hadn't decided to mentor me. He also helped me with this blog and some other KDE Community stuff.

Now, I will cover some information about my Season of KDE Project.

# My Season of KDE Project

## Overview

I will create a wrapper crate for KConfig KDE Framework in Season of KDE. It will allow the usage of KConfig from Rust projects without writing C++. The crate will be dependent on [qttypes](https://crates.io/crates/qttypes), which wraps a lot of QML types and makes them available in Rust. While I will mainly be testing it with [qmetaobject](https://crates.io/crates/qmetaobject), I would like to avoid having any hard dependency on it. My goal is to make the bindings safe while having as little overhead as possible.

Finally, I would like to find a way to have something similar to [KConfigXT](https://develop.kde.org/docs/use/configuration/kconfig_xt/) for Rust. This is a bit of a lofty goal, and I'm not entirely sure how to achieve it. However, I think KConfigXT is one of the best features of KConfig, and I don't believe the wrapper would be complete without it.

The kconfig bindings crate is available [here](https://invent.kde.org/oreki/kconfig-rs). Feel free to test it out and report Bugs.

## Goals

Creating bindings for more languages allows bringing more people to the KDE Community. Currently, Qt only has good support for C++ and Python. Rust promises to provide a safe and performant language with modern conveniences like a good package manager, async support, etc. As such, I think it can be pretty helpful to promote the use of Rust in KDE.

C++ is a great language, so I don't believe in rewriting everything in Rust. However, we should use it in places where it is better than C++. For example, Rust is excellent for parsers and thus can be used in KDevelop and Kate.

Currently, the barrier for using Rust in KDE is very high, which discourages most people from giving it a shot. This means that most Rust programmers never give KDE much of a chance, and in turn, there has been very little progress with Rust in KDE.

I want to create bindings for enough components that simple Kirigami applications can be written entirely in Rust. Since QML UI is already mostly decoupled from the C++ backend in most applications, writing the backend in Rust instead of C++ is not as difficult as trying to use QtWidgets from Rust. This should attract new application developers to consider Rust a viable option.

# Future

Making bindings between KDE and Rust shows surpringing contrasts between the two languages. In the next post, I'll explore how QtFlags can be used in Rust.

# Helpful Links for Rust/Qt Development

1. [qmetaobject](https://crates.io/crates/qmetaobject)
2. [Rust Qt Binding Generattor](https://invent.kde.org/sdk/rust-qt-binding-generator)
3. [ki18n crate](https://github.com/Ayush1325/ki18n-rs)
4. [kconfig crate](https://invent.kde.org/oreki/kconfig-rs)
5. [rust-qt-template](https://invent.kde.org/oreki/rust-qt-template)
6. [kirigami crate](https://github.com/Ayush1325/kirigami-rs)
7. [kde frameworks crate](https://github.com/Ayush1325/kde-frameworks-rs)
