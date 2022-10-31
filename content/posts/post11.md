+++
title = "Implementing Stdio for UEFI"
description = "An implementation of Stdio for UEFI"
date = "2022-07-08T00:53:12+05:30"

[taxonomies]
categories = ["post"]
tags = ["rust", "tianocore", "gsoc22", "uefi"]
+++

Hello everyone. In this post, I will go through how I implemented the stdio module for UEFI. The implementation code can be found [here](https://github.com/Ayush1325/rust/blob/7f367e14f687a7d485c1d3410e2cf5e22df8f1ad/library/std/src/sys/uefi/stdio.rs).

<!-- more -->
<br>

# UEFI Protocols
These are the protocols we will be using for implementing stdio. According to the spec, at least a blank implementation of these protocols should be present.

## Simple Text Input Protocols
### Interface Structure
```c
typedef struct _EFI_SIMPLE_TEXT_INPUT_PROTOCOL {
  EFI_INPUT_RESET Reset;
  EFI_INPUT_READ_KEY ReadKeyStroke;
  EFI_EVENT WaitForKey;
} EFI_SIMPLE_TEXT_INPUT_PROTOCOL;
```
### Description
This protocol allows taking console input one character at a time.

## Simple Text Output Protocol
### Interface Structure
```c
typedef struct _EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL {
  EFI_TEXT_RESET Reset;
  EFI_TEXT_STRING OutputString;
  EFI_TEXT_TEST_STRING TestString;
  EFI_TEXT_QUERY_MODE QueryMode;
  EFI_TEXT_SET_MODE SetMode;
  EFI_TEXT_SET_ATTRIBUTE SetAttribute;
  EFI_TEXT_CLEAR_SCREEN ClearScreen;
  EFI_TEXT_SET_CURSOR_POSITION SetCursorPosition;
  EFI_TEXT_ENABLE_CURSOR EnableCursor;
  SIMPLE_TEXT_OUTPUT_MODE *Mode;
} EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL;
```
### Description
This protocol allows writing UCS-2 strigs to console. This protocol will be used to implement both Stdout and Stderr.

## SystemTable
### Interface Structure
```c
typedef struct {
  EFI_TABLE_HEADER Hdr;
  CHAR16 *FirmwareVendor;
  UINT32 FirmwareRevision;
  EFI_HANDLE ConsoleInHandle;
  EFI_SIMPLE_TEXT_INPUT_PROTOCOL *ConIn;
  EFI_HANDLE ConsoleOutHandle;
  EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL *ConOut;
  EFI_HANDLE StandardErrorHandle;
  EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL *StdErr;
  EFI_RUNTIME_SERVICES *RuntimeServices;
  EFI_BOOT_SERVICES *BootServices;
  UINTN NumberOfTableEntries;
  EFI_CONFIGURATION_TABLE *ConfigurationTable;
} EFI_SYSTEM_TABLE;
```
### Description
SystemTable is passed at the entry function. It provides the pointers to access the Stdin, Stdout, and Stderr.

<br>

# Stdout
I am using the `ConOut` pointer from the SystemTable for Stdout implementation. The signature of `io::Write::write`:
```rust
fn write(&mut self, buf: &[u8]) -> io::Result<usize>;
```
As you can see, we are provided with a buffer containing a UTF-8 bytes slice. I have added some functions as a part of `std::sys_common::ucs2` that make the conversion of UTF-8 to UCS-2 pretty easy. It also replaces all the unsupported characters (4-byte UTF-8 characters) with the Unicode unsupported character (U+fffd).

Currently, I am using a fixed-length slice (length = 4096) for the UCS-2 string since this is what Windows and other OS seem to do. Technically, it is possible to use heap allocation here as well; I have just decided not to use allocation inside the standard library until I need to.

Finally, I am also converting LF (Rust default) to CRLF. This allows having the same behavior when using Rust printing macros as in other operating systems.

<br>

# Stderr
The implementation of Stderr is the same as Stdout. The only difference is that I use the `StdErr` pointer here.

<br>

# Stdin
I am using the `ConIn` pointer from SystemTable for Stdin implementation. The signature for `io::Read::read`:
```rust
fn read(&mut self, buf: &mut [u8]) -> io::Result<usize>;
```
As you can see, we are supplied with the buffer in which we have to write the UTF-8 formatted bytes. This function should return the number of bytes read. Since the Standard Input Protocol only provides reading a single keystroke, I simply read a single UCS-2 character each time (which can be multiple bytes) and return. This works fine for the most part. However, it does have one drawback; there is line editing support. This means currently, while reading, you cannot edit any mistakes. While it might seem like line editing works, that's just because it reads and stores the `backspace` symbol while reading. I might try implementing line editing at some point. However, for now, I think this single-character read implementation is acceptable.

**Note:** Reading and returning a single character from this method does not mean that the Rust `std::io` functions will stop after reading a single character. All the methods like `readline`, etc. work how they are supposed to.

I am also printing every character read to Stdout to provide feedback (since most OSs do this).

Finally, this function converts CR (`enter` seems to be read as CR in UEFI) to LF. This is needed because all the Rust io functions are designed around LF.

<br>

# Conclusion
This is just a brief overview of the implementation of stdio for UEFI. I tried not to put too much code here since I am making many changes, and the code becomes obsolete as soon as I publish the post. All my work for UEFI std can be found [here](https://github.com/Ayush1325/rust/tree/uefi-std-next). Feel free to test it out for yourself.
