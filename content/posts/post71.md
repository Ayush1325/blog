+++
title = "RISC-V Privilege Levels"
description = "Notes on the RISC-V privilege modes, written while adding Supervisor mode support to Zephyr for the BeagleV-Fire."
date = "2026-09-06T00:30:12+05:30"

[taxonomies]
tags = ["zephyr"]

[extra]
comment = true
+++

Hello everyone. It has been a while since my last post. Recently, I encountered some missing
functionality in Zephyr that led me down a rabbit hole into RISC-V privilege levels. While I 
was vageuly aware that CPUs have privilege levels, I never had any reason to get a deeper 
understanding regarding the subject. Since I ended up working closely on something related to it,
I figured I should document things for my future self.

# Background

[BeagleV-Fire](https://www.beagleboard.org/boards/beaglev-fire) is a SBC by BeagleBoard.org with
[Microchip® Polarfire® MPFS025T](https://www.microchip.com/en-us/product/MPFS025T) SoC. Unlike most
other BeagleBoard.org boards, the initial Zephyr support for it was 
[upstreamed](https://github.com/zephyrproject-rtos/zephyr/pull/66389) by 
[Conor Paxton](https://github.com/con-pax) rather than of me.

Recently, I started exploring Zephyr on BeagleV-Fire, and found that the only supported way to boot
a Zephyr image was to build a custom image and
[write it to the board's eMMC](https://docs.zephyrproject.org/latest/boards/beagle/beaglev_fire/doc/index.html#flashing).

On boards like [PocketBeagle 2](https://www.beagleboard.org/boards/pocketbeagle-2), I normally use
u-boot to launch the Zephyr firmware. Having the ability to use u-boot also allows loading firmware
over nfs or tftp, which makes development much faster. So I wanted a similar development setup for
BeagleV-Fire.

Initially, I assumed it would come down to a few devicetree and Kconfig changes. However, I soon
discovered that Zephyr simply lacked the ability to launch in Supervisory mode in RISC-V. It always
assumed that it would be entered in Machine Mode, even with `RISCV_S_MODE` set. All the option did
was to make Zephyr perform an M-mode to S-mode transition, and run a piece of code (a crude SBI
implementation) in M-Mode to service M-mode only functionality.

In this post, I will go over the different privilege modes and why they are required.

# Why have privilege levels at all

The idea behind privilege levels is simple: code that is more trusted gets to touch more of the
machine. An application should not be able to reprogram the interrupt controller. A kernel should
not be able to accidentally reconfigure the memory protection that firmware set up around it. Each
level down the stack is handed a smaller machine than the one above it, and the boundaries are
enforced by hardware rather than by convention.

There is a second, less obvious reason, and it is the one that matters most for this post:
*portability*. Everything that is genuinely SoC specific, or that vendors can get wrong, can be
kept in a small firmware blob at the highest privilege level. The OS below it talks to that firmware
through a stable interface instead of poking vendor registers directly. On RISC-V that interface is
the [Supervisory Binary Interface](https://docs.riscv.org/reference/sbi/v3.0/index.html).

# Privilege levels in RISC-V

RISC-V defines three privilege modes:

| Level | Encoding | Name             | Abbreviation |
| ----- | -------- | ---------------- | ------------ |
| 0     | `00`     | User/Application | U            |
| 1     | `01`     | Supervisor       | S            |
| 2     | `10`     | *Reserved*       |              |
| 3     | `11`     | Machine          | M            |

Higher encoding means higher privilege.

There is no CSR that tells you which mode you are currently running in. The current privilege level
is the internal state of the hart. Software is expected to already know, because in practice each
piece of software is *built* for a particular level.

## Machine mode (M-mode)

M-mode is the highest privilege level and the only mandatory one. Every RISC-V hart comes out of
reset in M-mode. Code here has unrestricted access to the entire physical address space, every CSR,
and all of the platform hardware. Nothing below it can constrain it, and there is no mechanism to
sandbox it.

This is where firmware lives: DDR and clock bring-up, errata workarounds, and the runtime that
serves the OS. Because M-mode cannot be restricted, the goal is to run as little as possible in it.

For a small MCU-class RTOS, M-mode is often the *only* mode implemented on the target, which is
exactly the world Zephyr was originally written for.

## Supervisor mode (S-mode)

S-mode is where an OS kernel runs: Linux, the BSDs, u-boot proper, and now Zephyr. It gets virtual
memory through `satp` and the MMU, its own set of trap handling CSRs, its own interrupt enable and
pending bits, and access to whatever physical memory M-mode decided to leave open to it.

What it does *not* get is any view of M-mode. S-mode cannot read `mhartid`, cannot write `mtvec`.
As far as S-mode is concerned, those registers do not exist.

S-mode is optional, and it is only meaningful together with U-mode. A system with S-mode but no
U-mode would have a kernel and nothing to isolate it from.

## User mode (U-mode)

The least privileged level, where application code runs. U-mode has no trap CSRs of its own, so
every trap it takes is handled by a higher level. Paired with the MMU under S-mode this is what
gives you process isolation.

## The CSRs

Most of the trap handling machinery exists twice, once per mode, with an `m` or `s` prefix:

| Purpose                    | M-mode                | S-mode        |
| -------------------------- | --------------------- | ------------- |
| Status                     | `mstatus`             | `sstatus`     |
| Trap vector                | `mtvec`               | `stvec`       |
| Exception PC               | `mepc`                | `sepc`        |
| Trap cause                 | `mcause`              | `scause`      |
| Interrupt enable/pending   | `mie` / `mip`         | `sie` / `sip` |
| Scratch                    | `mscratch`            | `sscratch`    |
| Address translation        | —                     | `satp`        |
| Hart ID                    | `mhartid`             | —             |

More CSRs exist, but I am only mentioning the ones I encountered during my work.

The two asymmetric entries are the interesting ones. `satp` only exists in S-mode, because M-mode
never uses address translation. And there is no `shartid`, so an S-mode payload cannot ask the
hardware which hart it is running on. Instead, the RISC-V boot protocol hands the payload
`a0 = hartid` and `a1 = DTB pointer`, and it is expected to hold on to them.

The access rule is worth stating explicitly, because it caused me some confusion. CSR address bits
`[9:8]` encode the lowest privilege level allowed to access that CSR. Attempting to read `mhartid`
from S-mode raises an illegal instruction exception. This is the reason why the Zephyr would just
throw an exception when I naively tried launching it from u-boot.

# Ending Thoughts

That is all for this post. In a future post I will cover how a hart actually moves between these
modes, the Supervisor Binary Interface (SBI), openSBI, and how S-mode code talks to the firmware
sitting above it. After that, I will come back to BeagleV-Fire and go through what it took to make
Zephyr boot as an S-mode payload from u-boot.

Consider [supporting me](@/_index.md#support-me) if you like my work.

# Helpful links

- [BeagleV-Fire](https://www.beagleboard.org/boards/beaglev-fire)
- [Microchip® Polarfire® MPFS025T](https://www.microchip.com/en-us/product/MPFS025T)
- [Initial Zephyr S-mode only PR](https://github.com/zephyrproject-rtos/zephyr/pull/113656)
- [The RISC-V Instruction Set Manual, Volume II: Privileged Architecture](https://docs.riscv.org/reference/isa/v20260120/priv/priv-index.html)
