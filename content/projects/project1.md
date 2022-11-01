+++
title = "Rust Std for UEFI"
description = "Implementation of Rust Standard Library for UEFI"
date = "2022-10-29T12:00:33+05:30"
weight = 1

[taxonomies]
categories = ["project"]
tags = ["rust", "uefi"]

[extra]
url="https://summerofcode.withgoogle.com/programs/2022/projects/PwQlcngc"
+++
This project was created as a part of Google Summer of Code 2022, under the Tianocore organization. It is a working implementation of Rust Standard Library for UEFI targets, with an open [upstream PR](https://github.com/rust-lang/rust/pull/100316).

<!-- more -->

# Mentors
This project would not have been possible without the guidance from my excellent mentors:
- [Michael Kubacki](mailto:Michael.Kubacki@microsoft.com)
- [Michael Kinney](mailto:michael.d.kinney@intel.com)

# Special Mentions
- [David Rheinsberg](https://github.com/dvdhrm): Author of [`r-efi`](https://github.com/r-efi) and related projects.

# Why use Std in UEFI
Since UEFI is a low-level target, some people might be wondering the actual usefulness of this work. While not all UEFI projects would benefit from having std support, here are a few examples:
1. Writing UEFI shell applications. This includes stuff like benchmarks, self-test utilities, etc. 
2. Finding UEFI target bugs. During this work, I have found 3 numeric tests that cause CPU exceptions for UEFI (they are fixed now. Also, I have found 2 additional bugs (which seem like bugs in llvm soft-float) which went unnoticed because there was no easy way to do any broad testing.
3. Provide a stable interface for library developers. The current std contains some functions under std::os::uefi::env to provide access to the underlying SystemTable and SystemHandle, which are essential for writing anything for UEFI. This can allow UEFI-related crates to add extra functionality if std is being used.
4. Now that the entry function has been fixed (see [Using Rust main from custom entry point (Part 3)](@/posts/post16.md)), many drivers can also use Std.


# Std Implementation Status
- [x] alloc
  - [x] GlobalAlloc
    - [x] alloc
    - [x] dealloc
- [x] args
  - [x] Args
  - [x] args: Implement using `EFI_LOADED_IMAGE_PROTOCOL`
- [x] cmath: Provided by `compiler_builtins` for UEFI.
- [x] env: Just contains some constants
- [ ] fs
  - [ ] File
    - [x] *open: Can only open files in the same volume as executable.
    - [x] file_attr
    - [x] *fsync: Uses `EFI_FILE_PROTOCOL.Flush()` internally
    - [x] *datasync: Uses fsync internally
    - [x] truncate
    - [x] read
    - [x] *read_vectored: Using rust default implementation.
    - [x] is_read_vectored
    - [x] write
    - [x] *write_vectored: Using rust default implementation.
    - [x] is_write_vectored
    - [x] flush: Don't really maintain any buffer in Rust side.
    - [x] seek
    - [ ] duplicate
    - [x] set_permissions
  - [x] FileAttr
    - [x] size
    - [x] perm
    - [x] file_type
    - [x] modified
    - [x] accessed
    - [x] created
  - [x] ReadDir
  - [x] DirEntry
    - [x] path
    - [x] file_name
    - [x] metadata
    - [x] file_type
  - [x] OpenOptions
    - [x] new
    - [x] read
    - [x] write
    - [x] append
    - [x] truncate
    - [x] create
    - [x] create_new
  - [x] FilePermissions
    - [x] readonly
    - [x] set_readonly
  - [x] FileType
    - [x] is_dir
    - [x] is_file
    - [x] *is_symlink: Just returns false since I don't think UEFI supports symlinks.
  - [x] DirBuilder
    - [x] new
    - [x] *mkdir: Can only open files in the same volume as executable.
  - [x] readdir
  - [x] unlink
  - [x] rename
  - [x] set_perm
  - [x] rmdir
  - [x] remove_dir_all
  - [x] try_exists
  - [ ] readlink
  - [ ] symlink
  - [ ] link
  - [x] stat
  - [x] lstat
  - [ ] canonicalize
  - [x] copy
- [x] io
    - [x] IoSlice
        - [x] new
        - [x] advance
        - [x] as_slice
    - [x] IoSliceMut
        - [x] new
        - [x] advance
        - [x] as_slice
        - [x] as_mut_slice
- [x] *locks: Using the default implementation at `unsupported/locks.rs`. It should work for any target having just a single-thread.
- [ ] *net: Only implmented TCPv4 right now.
  - [ ] TcpStream
    - [x] connect
    - [x] connect_timeout
    - [x] set_read_timeout
    - [x] set_write_timeout
    - [x] read_timeout
    - [x] write_timeout
    - [ ] peek
    - [x] read
    - [x] read_vectored
    - [x] is_read_vectored
    - [x] write
    - [x] write_vectored
    - [x] is_write_vectored
    - [x] peer_addr
    - [x] socket_addr
    - [x] *shutdown:  Only implemented complete shutdown right now.
    - [ ] duplicate
    - [ ] linger
    - [ ] set_nodelay
    - [x] nodelay
    - [ ] set_ttl
    - [x] ttl
    - [ ] take_error
    - [ ] set_nonblocking
  - [ ] TcpListener
    - [x] bind
    - [x] socket_addr
    - [x] accept
    - [ ] duplicate
    - [x] set_ttl
    - [x] ttl
    - [ ] set_only_v6
    - [ ] only_v6
    - [ ] take_error
    - [ ] set_nonblocking
  - [ ] UdpSocket
- [ ] os
  - [x] errno
  - [x] error_string
  - [x] getcwd: Returns the Text representation of `EFI_DEVICE_PATH_PROTOCOL`. This can be directly converted to `EFI_DEVICE_PATH_PROTOCOL` using the `EFI_DEVICE_PATH_FROM_TEXT_PROTOCOL`.
  - [ ] chdir
  - [ ] SplitPaths
  - [ ] split_paths
  - [ ] JoinPaths
  - [ ] join_paths
  - [x] current_exe: Returns the Text representation of `EFI_DEVICE_PATH_PROTOCOL`. This can be directly converted to `EFI_DEVICE_PATH_PROTOCOL` using the `EFI_DEVICE_PATH_FROM_TEXT_PROTOCOL`.
  - [x] Env
  - [x] env
  - [x] getenv
  - [x] setenv
  - [x] unsetenv
  - [ ] temp_dir
  - [ ] home_dir
  - [x] exit
  - [ ] getpid
- [x] os_str: Uses Windows implementation
- [x] path
  - [x] MAIN_SEP_STR
  - [x] MAIN_SEP
  - [x] is_sep_byte
  - [x] is_verbatim_sep
  - [x] parse_prefix: Since `std::path::Prefix` cannot be extended, this just returns `None`.
  - [x] absolute
- [ ] pipe
  - [ ] *AnonPipe: Implemented using custom protocol
    - [x] read
    - [x] read_vectored: Using default implementation
    - [x] is_read_vectored
    - [x] write
    - [x] write_vectored: Using default implementation
    - [x] is_write_vectored
    - [ ] diverge
  - [x] *read2: It is blocking and synchronous right now
- [ ] process
  - [ ] Command
    - [x] new
    - [x] arg
    - [x] env_mut
    - [ ] cwd
    - [x] stdin
    - [x] stdout
    - [x] stderr
    - [x] get_program
    - [x] get_args
    - [x] get_envs
    - [x] get_current_dir
    - [x] *spawn: Currently calling `EFI_BOOT_SERVICES.StartImage()` since the Rust Command API seems to assume that Pipes can be operated in parallel.
  - [x] StdioPipes
  - [x] Stdio
  - [x] ExitStatus
    - [x] exit_ok
    - [x] code
  - [x] ExitStatusError
    - [x] code
  - [x] ExitCode
    - [x] as_i32
  - [ ] Process
    - [ ] id
    - [ ] kill
    - [x] wait
    - [x] try_wait
  - [ ] CommandArgs
- [x] stdio
  - [x] STDIN_BUF_SIZE
  - [x] Stdin
    - [x] new
    - [x] *read: Buffered reading not implemented yet.
  - [x] Stdout
    - [x] new
    - [x] write
    - [x] flush
  - [x] Stderr
    - [x] new
    - [x] write
    - [x] flush
- [ ] thread
    - [ ] Thread
        - [ ] new
        - [ ] yield_now
        - [ ] set_name
        - [x] sleep
        - [ ] join
    - [x] available_parallelism: Just returns 1 since UEFI is single-threaded
- [ ] thread_local_key: Using `unsupported/thread_local_key.rs`
- [x] time
  - [x] Instant
    - [x] now
    - [x] checked_sub_instant
    - [x] checked_add_duration
    - [x] checked_sub_duration
  - [x] SystemTime: Implemented using `GetTime()`
    - [x] now
    - [x] sub_time
    - [x] checked_add_duration
    - [x] checked_sub_duration

# Conclusion
I will continue to work on improving UEFI development in Rust in my spare time. Feel free to check out the GitHub PR if you are interested in this project. Also, consider supporting me if you like my work.

# Helpful Links
- [GSoC22: Summary Post](@/posts/post15.md)
- [Upstream PR](https://github.com/rust-lang/rust/pull/100316)
- [Tianocore Rust Repository](https://github.com/tianocore/rust/tree/uefi-master)
- [GSoC22 Archive Link](https://summerofcode.withgoogle.com/programs/2022/projects/PwQlcngc)