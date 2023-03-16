+++
title = "GSoC 2022: Progress Report 2"
description = "Google Summer of Code Progress Report 2"
date = "2022-07-24T02:38:12+05:30"

[taxonomies]
categories = ["post"]
tags = ["rust", "tianocore", "gsoc22", "uefi"]
+++
Hello everyone. It is finally possible to run the whole Rust testing suit for UEFI under QEMU and OVMF. So I think this is a good point to give a detailed overview of everything that has been implemented and the state of those implementations. I will also provide instructions on how to go about running the tests as well.

<!-- more -->
<br>

# Running Rust Tests
## Setup
We will follow the instructions in [Running Tests in a Remote Machine](https://rustc-dev-guide.rust-lang.org/tests/running.html#running-tests-on-a-remote-machine).
1. Clone my fork of Rust source and switch to `uefi-std-rebase` branch:
```sh
git clone https://github.com/Ayush1325/rust.git
cd rust
git checkout uefi-std-rebase
```
2. Make sure that all the dependencies have been met. Try building std for UEFI:
```sh
./x.py build --stage 1 --target x86_64-unknown-uefi
```
3. Build `remote-test-server`:
```sh
./x.py build --stage 1 --target x86_64-unknown-uefi src/tools/remote-test-server
```
## Launch QEMU 
Now we have to launch the generated `build/x86_64-unknown-linux-gnu/stage1-tools-bin/remote-test-server.efi` in QEMU. The following options are needed for launching this executable:
1. Network with port 12345 forwarded 
```sh
-netdev user,id=net0,hostfwd=tcp::12345-:12345 -device virtio-net-pci,netdev=net0,mac=00:00:00:00:00:00
```
2. OVMF:
```sh
-drive if=pflash,format=raw,file={Path to OVMF_CODE.fd},index=0 -drive if=pflash,format=raw,file={Path to OVMF_VARS.fd},index=1
```
3. UEFI Shell: Not supplying this ISO makes OVMF try to boot from the network first for me, so I do this manually as well:
```sh
-drive format=raw,file={Path to UefiShell.iso},index=2
```
4. Image containing the executable and a `startup.nsh` (for automatic startup): I would recommend not using `vvfat` since it gives a lot of mapping errors in intensive File I/O which is needed in running the tests

All of this can be simplified by using my fork of [uefi-run](https://github.com/Ayush1325/uefi-run/tree/uefi). If you are using that, then it is possible to use the config I am using currently:
```sh
qemu-uefi build/x86_64-unknown-linux-gnu/stage1-tools/x86_64-unknown-uefi/release/remote-test-server.efi -s 1024 --shell-path UefiShell.iso --vars-path OVMF_VARS.fd -b OVMF_CODE.fd --output-path output.txt -- -netdev user,id=net0,hostfwd=tcp::12345-:12345 -device virtio-net-pci,netdev=net0,mac=00:00:00:00:00:00
```
The output-path argument passes `-serial output.txt` since the stderr output is not visible on the VGA buffer for me. However, the output.txt is extremely useful for debugging since all the stderr (Rust panics and aborts) are present in the file with the correct location of panics.

I would also recommend launching the executable with `verbose` argument.

## Run test
Any of the Rust test/s or test folders can be run on the QEMU instance using the following command:
```sh
TEST_DEVICE_ADDR="localhost:12345" ./x.py test src/test/ui/{FILE or Directory} --target x86_64-unknown-uefi --stage 1
```

<br>

# Std Implementation Status
- [x] alloc
  - [x] GlobalAlloc: Currently uses hardcoded `EFI_MEMORY_TYPE::EfiLoaderData`. Can be changed.
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
    - [ ] truncate
    - [x] read
    - [x] *read_vectored: Using rust default implementation.
    - [x] is_read_vectored
    - [x] write
    - [x] *write_vectored: Using rust default implementation.
    - [x] is_write_vectored
    - [x] flush: Don't really maintain any buffer in Rust side.
    - [x] seek
    - [ ] duplicate
    - [ ] set_permissions
  - [x] FileAttr
    - [x] size
    - [x] perm
    - [x] file_type
    - [x] modified
    - [x] accessed
    - [x] created
  - [x] ReadDir
  - [ ] DirEntry
    - [ ] path
    - [x] file_name
    - [x] metadata
    - [x] file_type
  - [ ] OpenOptions
    - [x] new
    - [x] read
    - [x] write
    - [x] append
    - [ ] truncate
    - [x] create
    - [ ] create_new
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
  - [ ] rename
  - [ ] set_perm
  - [x] rmdir
  - [x] remove_dir_all
  - [ ] try_exists
  - [ ] readlink
  - [ ] symlink
  - [ ] link
  - [x] stat
  - [x] lstat
  - [ ] canonicalize
  - [ ] copy
- [x] *io: Using the default implementation at `unsupported/io.rs`. It works fine.
- [x] *locks: Using the default implementation at `unsupported/locks.rs`. It should work for any target having just a single-thread.
- [ ] *net: Only implmented TCPv4 right now.
  - [ ] TcpStream
    - [ ] connect
    - [ ] connect_timeout
    - [ ] set_read_timeout
    - [ ] set_write_timeout
    - [ ] read_timeout
    - [ ] write_timeout
    - [ ] peek
    - [x] read
    - [x] *read_vectored: Using the default implementation right now. However, EFI_TCP4_PROTOCOL supports vectored read.
    - [x] is_read_vectored
    - [x] write
    - [x] *write_vectored: Using the default implementation right now. However, EFI_TCP4_PROTOCOL supports vectored write.
    - [x] is_write_vectored
    - [ ] peer_addr
    - [ ] socket_addr
    - [ ] *shutdown:  Only implemented complete shutdown right now.
    - [ ] duplicate
    - [ ] linger
    - [ ] set_nodelay
    - [ ] nodelay
    - [ ] set_ttl
    - [ ] ttl
    - [ ] take_error
    - [ ] set_nonblocking
  - [ ] TcpListener
    - [x] bind
    - [ ] socket_addr
    - [x] accept
    - [ ] duplicate
    - [ ] set_ttl
    - [ ] ttl
    - [ ] set_only_v6
    - [ ] only_v6
    - [ ] take_error
    - [ ] set_nonblocking
  - [ ] UdpSocket
- [ ] os
  - [x] errno
  - [ ] error_string
  - [x] getcwd: Returns the Text representation of `EFI_DEVICE_PATH_PROTOCOL`. This can be directly converted to `EFI_DEVICE_PATH_PROTOCOL` using the `EFI_DEVICE_PATH_FROM_TEXT_PROTOCOL`.
  - [ ] chdir
  - [ ] SplitPaths
  - [ ] split_paths
  - [ ] JoinPaths
  - [ ] join_paths
  - [x] current_exe: Returns the Text representation of `EFI_DEVICE_PATH_PROTOCOL`. This can be directly converted to `EFI_DEVICE_PATH_PROTOCOL` using the `EFI_DEVICE_PATH_FROM_TEXT_PROTOCOL`.
  - [ ] Env
  - [ ] env
  - [x] *getenv: Currently using a static Guid. Maybe should use the Guid used by UefiShell for Environment Variables.
  - [x] setenv
  - [x] unsetenv
  - [ ] temp_dir
  - [ ] home_dir
  - [x] exit
  - [ ] getpid
- [x] os_str: Basically just UTF-8 strings. This is because os_str is supposed to be the superset of both te OS specific string(UCS-2) and Rust strings (UTF-8).
  - [x] Buf
    - [x] from_string
    - [x] with_capacity
    - [x] clear
    - [x] capacity
    - [x] reserve
    - [x] try_reserve
    - [x] reserve_exact
    - [x] try_reserve_exact
    - [x] shrink_to_fit
    - [x] shrink_to
    - [x] as_slice
    - [x] as_mut_slice
    - [x] into_string
    - [x] push_slice
    - [x] into_box
    - [x] into_arc
    - [x] into_rc
  - [x] Slice
    - [x] from_u8_slice
    - [x] from_str
    - [x] to_str
    - [x] to_string_lossy
    - [x] to_owned
    - [x] clone_into
    - [x] into_box
    - [x] empty_box
    - [x] into_arc
    - [x] into_rc
    - [x] make_ascii_lowercase
    - [x] make_ascii_uppercase
    - [x] to_ascii_lowercase
    - [x] to_ascii_uppercase
    - [x] is_ascii
    - [x] eq_ignore_ascii_case
- [ ] path
  - [x] MAIN_SEP_STR
  - [x] MAIN_SEP
  - [x] is_sep_byte
  - [x] is_verbatim_sep
  - [ ] parse_prefix
  - [ ] absolute
- [ ] pipe
  - [ ] *AnonPipe: Implemented using Runtime Variables
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
    - [ ] *stdin: Only null working yet.
    - [x] *stdout: Using my primitive AnonPipe
    - [x] *stderr: Using my primitive AnonPipe
    - [x] get_program
    - [x] get_args
    - [x] get_envs
    - [x] get_current_dir
    - [x] *spawn: Currently calling `EFI_BOOT_SERVICES.StartImage()` here since the pipes don't really work asynchronously right now.
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
    - [ ] try_wait
  - [ ] CommandArgs
- [x] stdio
  - [x] *STDIN_BUF_SIZE: Currently using the same value as Windows
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
- [ ] thread: Using `unsupported/thread.rs`
- [ ] thread_local_key: Using `unsupported/thread_local_key.rs`
- [ ] time
  - [ ] Instant
  - [x] SystemTime: Implemented using `GetTime()`
    - [x] now
    - [x] sub_time
    - [x] checked_add_duration
    - [x] checked_sub_duration

<br>

# Conclusion
Since the first main object to run Rust tests has now been met, I will work on improving/refactoring most of the implementations before trying to merge them to master. It would also be great if more people try out this std and give feedback.

Consider [supporting me](@/pages/supportme.md) if you like my work.
