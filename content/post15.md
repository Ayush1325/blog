+++
title = "GSoC22: Final Blog Post"
description = "A blog post summarizing all my Google Summer of Code 2022 Work"
date = "2022-09-05T22:22:12+05:30"

[taxonomies]
tags = ["rust", "tianocore", "gsoc22", "uefi"]
+++
Hello everyone. This is probably the final blog post I write before Google Summer of Code 2022 ends (hopefully not the final uefi one). In this post, I will summarize my GSoC22 work to make it more accessible to everyone.

<!-- more -->

<br>

# PRs
Just a list of all PRs I opened as a part of GSoC22. While other people created some additional PRs to fix problems I found during my work; I do not include them since I wasn't the one who found the solution for those.

## r-efi
1. [Getting r-efi ready for use in Rust std](https://github.com/r-efi/r-efi/pull/33): 4 commits
2. [Add TCP6 Protocol](https://github.com/r-efi/r-efi/pull/34): 1 commit
3. [Add EFI_RNG_PROTOCOL](https://github.com/r-efi/r-efi/pull/36): 1 commit
4. [Add EFI_TIMESTAMP_PROTOCOL](https://github.com/r-efi/r-efi/pull/37): 1 commit
5. [Derive Default for Time](https://github.com/r-efi/r-efi/pull/38): 1 commit
6. [Add TCP4 and IP4 protocols](https://github.com/r-efi/r-efi/pull/39): 1 commit
7. [Fix MODE_CREATE value](https://github.com/r-efi/r-efi/pull/40): 1 commit
8. [Make NotifyFunction Optional](https://github.com/r-efi/r-efi/pull/42): 1 commit
9. [Add ICMP Error Definitions](https://github.com/r-efi/r-efi/pull/44): 1 commit
10. [Add `EFI_DEVICE_PATH_FROM_TEXT_PROTOCOL` and `EFI_DEVICE_PATH_TO_TEXT_PROTOCOL`](https://github.com/r-efi/r-efi/pull/45): 2 commits
11. [Make RNG protocol members public](https://github.com/r-efi/r-efi/pull/48): 1 commit
12. [Fix timestamp properties](https://github.com/r-efi/r-efi/pull/49): 1 commit
13. [Implement Default for tcp4::ConfigData](https://github.com/r-efi/r-efi/pull/50): 1 commit
14. [Add EFI_SHELL_PROTOCOL](https://github.com/r-efi/r-efi/pull/51): 1 commit. Still Open.
15. [Add UDP Protocols](https://github.com/r-efi/r-efi/pull/52): 2 commit. Still Open.

## compiler-builtins
1. [Enable mem for UEFI](https://github.com/rust-lang/compiler-builtins/pull/473): 1 commit
2. [Use all of src/math for UEFI](https://github.com/rust-lang/compiler-builtins/pull/480): 1 commit

## rust
- [Add Rust std support for x86_64-unknown-uefi](https://github.com/rust-lang/rust/pull/100316): 76 commits. Still Open.

<br>

# Blog Posts
1. [Google Summer of Code 2022](@/post5.md)
2. [Use Restricted std in UEFI](@/post6.md)
3. [Using Rust main from a custom entry point](@/post7.md)
4. [Writing an Allocator for UEFI](@/post8.md)
5. [Using Rust main from a custom entry point (Part 2)](@/post9.md)
6. [GSoC 2022: Progress Report 1](@/post10.md)
7. [Implementing Stdio for UEFI](@/post11.md)
8. [GSoC 2022: Progress Report 2](@/post12.md)
9. [UEFI Rust std has a new home](@/post13.md)
10. [Writing UEFI Protocol in Rust](@/post14.md)

# Conclusion
I will continue working on getting Rust std [PR](https://github.com/rust-lang/rust/pull/100316) merged since it is still unmerged. I also have some more UEFI Rust Projects lined up, and I think I will keep working on them for the foreseeable future. Feel free to check out and experiment with using Rust for UEFI (with or without std).
