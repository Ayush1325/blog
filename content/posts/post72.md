+++
title = "RISC-V Supervisor Binary Interface"
description = "Notes on the RISC-V SBI specification, written while adding SMP support to Zephyr for S-mode payloads."
date = "2026-09-13T00:30:12+05:30"

[taxonomies]
tags = ["zephyr"]

[extra]
comment = true
+++

Hello everyone. This is a continuation of my [previous post](@/posts/post71.md) on RISC-V privilege
levels. Last time, I went over what the three modes are and which CSRs belong to which. This ended
on a problem: S-mode payload (kernel) does not have access to start/stop other cores, timers, etc.
The privilege split takes those things away, and something has to provide them back.

That something is Supervisor Binary Interface.

# What is SBI?

Supervisor Binary Interface (SBI) is a specification, maintained by RISC-V International alongside
the ISA specs that describes a set of services anything running in M-mode promises to provide to 
the S-mode payload. At the time of writing, the latest version is
[v3.0](https://docs.riscv.org/reference/sbi/v3.0/index.html).

[OpenSBI](https://github.com/riscv-software-src/opensbi) is an open source implementation of SBI 
Specification. A kernel does not target OpenSBI (or any other specific implementation); it targets
the specification, and any conforming implementation can host it.

# Why SBI is needed?

The privilege split in the previous post buys isolation, but on it's own it would be a portability
disaster. Every kernel would need to know which SoC it was on, how to start secondary cores,
timers, etc.

SBI moves all of that behind a fixed API. The vendor-specific knowledge lives in M-mode firmware. As
long as kernel and M-mode firmware support the same SBI specification version, no vendor-specific
code is required in kernel.

# Binary Encoding

All SBI functions share a single binary encoding, which facilitates the mixing of SBI extensions. The SBI
specification follows the below calling convention.

- An `ecall` is used as the control transfer instruction between the supervisor and the SEE.
- `a7` encodes the SBI extension ID (EID).
- `a6` encodes the SBI function ID (FID) for a given extension ID encoded in a7 for any SBI
extension defined in or after SBI v0.2.
- `a0` through `a5` contain the arguments for the SBI function call. Registers that are not defined
in the SBI function call are not reserved.
- All registers except `a0` & `a1` must be preserved across an SBI call by the callee.
- SBI functions must return a pair of values in `a0` and `a1`, with a0 returning an error code. This
is analogous to returning the C structure

```c
struct sbiret {
    long error;
    union {
        long value;
        unsigned long uvalue;
    };
};
```

Data type `long` in C pseudocode is `XLEN` bits wide.

`error` is zero on success and negative otherwise:

| Value | Name                        | Meaning                          |
| ----- | --------------------------- | -------------------------------- |
| 0     | `SBI_SUCCESS`               | Completed                        |
| -1    | `SBI_ERR_FAILED`            | Failed for an unspecified reason |
| -2    | `SBI_ERR_NOT_SUPPORTED`     | Extension or function is absent  |
| -3    | `SBI_ERR_INVALID_PARAM`     | Bad argument                     |
| -4    | `SBI_ERR_DENIED`            | Not permitted                    |
| -5    | `SBI_ERR_INVALID_ADDRESS`   | Address is not usable            |
| -6    | `SBI_ERR_ALREADY_AVAILABLE` | Already available                |
| -7    | `SBI_ERR_ALREADY_STARTED`   | Already started                  |
| -8    | `SBI_ERR_ALREADY_STOPPED`   | Already stopped                  |
| -9    | `SBI_ERR_NO_SHMEM`          | Shared memory not available      |
| -10   | `SBI_ERR_INVALID_STATE`     | Wrong state for this call        |
| -11   | `SBI_ERR_BAD_RANGE`         | Range outside what is allowed    |
| -12   | `SBI_ERR_TIMEOUT`           | Timed out                        |
| -13   | `SBI_ERR_IO`                | Input/output error               |

# SBI extensions

The SBI specification has the base extension that is mandatory, while everything else (IPI, timers,
hart management) is optional. The caller is expected to query if an extension is supported before
trying to use it.

Discovery goes through the Base extension (EID `0x10`):

| FID | Call                     | What it tells you                            |
| --- | ------------------------ | -------------------------------------------- |
| 0   | `sbi_get_spec_version`   | Which version of the spec you are talking to |
| 1   | `sbi_get_impl_id`        | OpenSBI, RustSBI, HSS, KVM…                  |
| 2   | `sbi_get_impl_version`   | Version of that implementation               |
| 3   | `sbi_probe_extension`    | Zero if the extension is absent              |
| 4–6 | `sbi_get_mvendorid` etc. | M-mode identification CSRs, handed down      |

FIDs 4 through 6 are the pattern in miniature: three M-mode CSRs that S-mode cannot read, exposed as
calls. There is deliberately no `sbi_get_mhartid` — the boot protocol already put it in `a0` and you
were supposed to keep it.

`sbi_get_impl_id` is worth knowing: `1` is OpenSBI, `4` is RustSBI, and `8` is Microchip's PolarFire
Hart Software Services, which is the one answering on the BeagleV-Fire.

Now I will go over some of the extensions that I directly had to interact with during my Zephyr work.

## Hart State Management Extension

My initial [Zephyr PR](https://github.com/zephyrproject-rtos/zephyr/pull/113656) made it possible to
have single-core setups start directly in S-mode. However, SMP setups (multi-core SoCs) were still
broken. What Zephyr required was a way to bring up secondary cores after the primary core is up and
running.

In M-mode, Zephyr does this itself. Every hart comes out of reset running the same image, the ones 
that are not the boot hart park themselves in a spin loop watching a flag in memory, and the boot 
hart writes that flag once the kernel is ready for them.

In S-mode, the firmware (u-boot) hands off exactly one hart to the running image. The rest are still
held by M-mode firmware. The only way to reach them is to ask firmware, which is what the Hart State
Management (HSM) extension is for. It sits at EID 0x48534D.

| FID | Function              | Purpose                                             |
| --- | --------------------- | --------------------------------------------------- |
| 0   | `sbi_hart_start`      | Start a stopped hart at a given address             |
| 1   | `sbi_hart_stop`       | Stop the *calling* hart. Does not return on success |
| 2   | `sbi_hart_get_status` | Query the state of a hart                           |
| 3   | `sbi_hart_suspend`    | Put the calling hart into a low power state         |

The one that matters for bring-up is:

```c
struct sbiret sbi_hart_start(unsigned long hartid,
                             unsigned long start_addr,
                             unsigned long opaque);
```

- `hartid`: specifies the target hart which is to be started.
- `start_addr`: points to a runtime-specified physical address, where the hart can start executing 
in supervisor-mode.
- `opaque`: an XLEN-bit value which will be set in the a1 register when the hart starts executing at
start_addr.

It is important to note that a hart has more states than just started and stopped:

```
STOPPED -> START_PENDING -> STARTED -> STOP_PENDING -> STOPPED
```

So `sbi_hart_start` returning `SBI_SUCCESS` does not mean the hart is running. It means firmware
accepted the request. The target transitions through `START_PENDING` on its own schedule, and the
caller has to synchronize with the target itself rather than trusting the return value. There are
matching `SUSPENDED`, `SUSPEND_PENDING` and `RESUME_PENDING` states for `sbi_hart_suspend`.

Zephyr already had the right abstraction for this: `pm_cpu_ops`. It is also used by ARM's PSCI.
The SBI side drops in as just another `pm_cpu_ops` driver (`CONFIG_PM_CPU_OPS_SBI`) instead of
something RISC-V specific bolted onto the SMP code.

## IPI Extension

Once the secondary cores are running, the kernel needs a way to interrupt them. Zephyr uses IPIs to
tell another core to reschedule, and on FPU-enabled builds to make a core flush its floating point
context.

In M-mode this is just a store. The CLINT has a `msip` register per hart, and writing to it raises a
machine software interrupt on that hart. From S-mode the CLINT belongs to M-mode, so this becomes
SBI's problem too. The IPI extension lives at EID `0x735049` and has exactly one function:

```c
struct sbiret sbi_send_ipi(unsigned long hart_mask,
                           unsigned long hart_mask_base);
```

- `hart_mask`: A scalar bit-vector containing hartids
- `hart_mask_base`: The starting hartid from which the bit-vector must be computed.

**NOTE:** In a single SBI function call, the maximum number of harts that can be set is always XLEN.
If a lower privilege mode needs to pass information about more than XLEN harts, it must invoke the
SBI function multiple times. hart_mask_base can be set to -1 to indicate that hart_mask shall be
ignored and all available harts must be considered.

This raises a *supervisor* software interrupt on the selected harts, which the receiving hart
handles through `stvec` and acknowledges by clearing `sip.SSIP`.

There is a knock-on effect worth flagging, since it caught a test rather than the code. SBI IPIs
arrive as the supervisor software interrupt, so `SSOFT` is now owned by the kernel's IPI handling and
is no longer available to anything else. The ISR table test assumed it was free and had to be
excluded for these configurations.

Both of these, along with the SMP plumbing around them, are in
[this PR](https://github.com/zephyrproject-rtos/zephyr/pull/117751).

# Ending Thoughts

In the next post I will get back to the BeagleV-Fire itself and go through what all of this meant in
practice: the boot header, hart ID handling, and finally loading a Zephyr image over tftp from
u-boot.

Consider [supporting me](@/_index.md#support-me) if you like my work.

# Helpful links

- [BeagleV-Fire](https://www.beagleboard.org/boards/beaglev-fire)
- [Microchip® Polarfire® MPFS025T](https://www.microchip.com/en-us/product/MPFS025T)
- [RISC-V Supervisor Binary Interface Specification v3.0](https://docs.riscv.org/reference/sbi/v3.0/index.html)
- [Zephyr S-mode only PR](https://github.com/zephyrproject-rtos/zephyr/pull/113656)
- [Zephyr RISC-V SMP over SBI PR](https://github.com/zephyrproject-rtos/zephyr/pull/117751)
- [OpenSBI](https://github.com/riscv-software-src/opensbi)
- [The RISC-V Instruction Set Manual, Volume II: Privileged Architecture](https://docs.riscv.org/reference/isa/v20260120/priv/priv-index.html)
