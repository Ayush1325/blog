+++
title = "Writing an Allocator for UEFI"
description = "How to write a system allocator for UEFI"
date = "2022-06-23T16:45:12+05:30"

[taxonomies]
categories = ["post"]
tags = ["rust", "tianocore", "gsoc22", "uefi"]
+++

Hello everyone, we will discuss how to write the Allocator for UEFI in this post. More specifically, the Allocator is for the DEX phase in UEFI. We will be using [uefi-spec-rs](https://github.com/Ayush1325/uefi-spec-rs), which is my wrapper around [r-efi](https://github.com/Ayush1325/r-efi) for use in the std.

<!-- more -->
<br>

# UEFI Memory Memory Management Services
First, let us look at the Memory Management Services we will be using in UEFI.
## AllocatePool
### Prototype
```c
typedef
EFI_STATUS
(EFIAPI *EFI_ALLOCATE_POOL) (
  IN EFI_MEMORY_TYPE PoolType,
  IN UINTN Size,
  OUT VOID **Buffer
);
```
### Description
This function allocates a memory region of `Size` bytes from the memory of type `PoolType` and returns the address of the allocated memory in the location referenced by Buffer. This function allocates pages from EfiConventionalMemory as needed to grow the requested pool type. 

**Note:** All allocations are eight-byte aligned.

### Status Codes Returned
1. **EFI_SUCCESS** : The requested number of bytes was allocated.
2. **EFI_OUT_OF_RESOURCES** : The pool requested could not be allocated.
3. **EFI_INVALID_PARAMETER** : PoolType is in the range EfiMaxMemoryType..0x6FFFFFFF.
4. **EFI_INVALID_PARAMETER** : PoolType is EfiPersistentMemory.
5. **EFI_INVALID_PARAMETER** : Buffer is NULL.

## FreePool
### Prototype
```c
typedef
EFI_STATUS
(EFIAPI *EFI_FREE_POOL) (
  IN VOID *Buffer
);
```
### Description
This function returns the memory specified by Buffer to the system. In return, the memory’s type is EfiConventionalMemory. The freed Buffer must have been allocated by `AllocatePool()`.

### Status Code Returned
1. **EFI_SUCCESS :**  The memory was returned to the system.
2. **EFI_INVALID_PARAMETER :** Buffer was invalid.

<br>

# Writing a basic allocator
First, we will write an allocator which works for alignment <= 8 bytes. To make a global allocator, we need to implement the' GlobalAlloc' trait. In the case of std, we would implement this trait on the `System` allocator. However, since I would like to test things out, I will be implementing the allocator as an example in the uefi-spec-rs crate.

```rust
static mut GLOBAL_SYSTEM_TABLE: GlobalData<efi::SystemTable> = GlobalData::new();

#[global_allocator]
static GLOBAL_ALLOCATOR: Allocator = Allocator;

struct Allocator;

unsafe impl GlobalAlloc for Allocator {
    unsafe fn alloc(&self, layout: core::alloc::Layout) -> *mut u8 {
      todo!()
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: core::alloc::Layout) {
      todo!()
    }
}
```
Here we create an empty struct (`Allocator`) as a placeholder for `System` and implement GlobalAlloc on it. The `GLOBAL_SYSTEM_TABLE` stores an `AtomicPtr` to the `SystemTable`. I will also be adding a way to access the SystemTable in `std::os::uefi`. However, I will not be exposing the static mutable variable there.

## Implement alloc
Firstly, we need to be aware of a few things:
1. The `layout.size()` supplied in `GlobalAlloc` can be 0. It is up to the implementation to check for that case. 
2. According to docs, we should return the `NULL` pointer in case of errors. 
3. The `alloc` function should not unwind.

Keeping all that in mind, here is the basic implementation of alloc:
```rust
unsafe fn alloc(&self, layout: core::alloc::Layout) -> *mut u8 {
    let st = unsafe {
        match GLOBAL_SYSTEM_TABLE.get_mut() {
            Ok(x) => x,
            Err(_) => return core::ptr::null_mut(),
        }
    };

  
    let mut ptr: *mut core::ffi::c_void = core::ptr::null_mut();
    let align = layout.align();
    let size = layout.size();
   
    if size == 0 || align > 8 {
        return core::ptr::null_mut();
    }
   
    let r = memory_allocation_services::allocate_pool(st, efi::LOADER_DATA, size, &mut ptr);
   
    if r.is_err() || ptr.is_null() {
        return core::ptr::null_mut();
    }
   
    ptr.cast()
}
```
This is pretty simple. We fail for alignment > 0. we also fail if the `ptr` is null after allocation or if the error status is returned.

## Implement dealloc
We need to check that size is non-zero. The only error that `FreePool()` returns is in the case of invalid `ptr`, so ideally, this should not happen in `System` allocator. However, I am adding the assert for now to improve debugging. That assert will probably be removed in production.
```rust
unsafe fn dealloc(&self, ptr: *mut u8, layout: core::alloc::Layout) {
    let st = unsafe {
        match GLOBAL_SYSTEM_TABLE.get_mut() {
            Ok(x) => x,
            Err(_) => return core::ptr::null_mut(),
        }
    };


    if layout.size() != 0 {
        let r = memory_allocation_services::free_pool(st, ptr.cast());
        assert!(r.is_ok());
    }
}
```

## Testing the new allocator
We just add a `efi_main` function that initializes the `GLOBAL_SYSTEM_TABLE` and we should be good to use the `alloc` types:
```rust
pub fn efi_run() -> efi::Status {
    let st = unsafe {
        match GLOBAL_SYSTEM_TABLE.get_mut() {
            Ok(x) => x,
            Err(_) => return efi::Status::ABORTED,
        }
    };

    let s: String;
    let mut v: Vec<u16>;

    // Create string and convert to UTF-16. We need a terminating NUL, since UEFI uses C-String
    // style wide-strings.
    s = String::from("Hello World!\n");
    v = s.encode_utf16().collect();
    v.push(0);

    // Print the string on console-out.
    let r = simple_text_output::output_string(st, v.as_mut_slice());
    if r.is_err() {
        efi::Status::ABORTED
    } else {
        efi::Status::SUCCESS
    }
}

#[export_name = "efi_main"]
pub extern "C" fn main(_h: efi::Handle, st: *mut efi::SystemTable) -> efi::Status {
    let r = unsafe { GLOBAL_SYSTEM_TABLE.init(st) };
    if r.is_err() {
        return efi::Status::ABORTED;
    }

    efi_run()
}
```
This example works as expected.

<br>

# Getting allocations > 8-byte align to work
While the previous allocator works for many cases, there is a clever way to work around the 8-byte alignment limit. This is used in the [windows allocator](https://github.com/rust-lang/rust/blob/10f4ce324baf7cfb7ce2b2096662b82b79204944/library/std/src/sys/windows/alloc.rs) as well as [r-efi-alloc](https://github.com/Ayush1325/r-efi-alloc/blob/d47b50c75b6f16a44364f21f40db7a0f3d4dd296/src/alloc.rs).

## Implement alloc
The new `alloc` function looks like this:
```rust
unsafe fn alloc(&self, layout: core::alloc::Layout) -> *mut u8 {
    let st = unsafe {
        match GLOBAL_SYSTEM_TABLE.get_mut() {
            Ok(x) => x,
            Err(_) => return core::ptr::null_mut(),
        }
    };

    let align = layout.align();
    let size = layout.size();

    if size == 0 {
        return core::ptr::null_mut();
    }

    let mut ptr: *mut core::ffi::c_void = core::ptr::null_mut();
    let aligned_size = align_size(size, align);

    let r =
        memory_allocation_services::allocate_pool(st, efi::LOADER_DATA, aligned_size, &mut ptr);

    if r.is_err() || ptr.is_null() {
        return core::ptr::null_mut();
    }

    unsafe { align_ptr(ptr.cast(), align) }
}
```
The magic happens in the `align_size` and the `align_ptr` functions.

In `align_size` function, we just allocate extra padding in case `align` is greater than 8. This padding will later be used in the `align_ptr` function.
```rust
fn align_size(size: usize, align: usize) -> usize {
    if align > POOL_ALIGNMENT {
        // Allocate extra padding in order to be able to satisfy the alignment.
        size + align
    } else {
        size
    }
}
```

In the `align_ptr` function we store the original address as `Header` in front of the aligned_ptr. 
```rust
#[repr(C)]
struct Header(*mut u8);

unsafe fn align_ptr(ptr: *mut u8, align: usize) -> *mut u8 {
    if align > POOL_ALIGNMENT {
        let offset = align - (ptr.addr() & (align - 1));

        // SAFETY: `MIN_ALIGN` <= `offset` <= `layout.align()` and the size of the allocated
        // block is `layout.align() + layout.size()`. `aligned` will thus be a correctly aligned
        // pointer inside the allocated block with at least `layout.size()` bytes after it and at
        // least `MIN_ALIGN` bytes of padding before it.
        let aligned = unsafe { ptr.add(offset) };

        // SAFETY: Because the size and alignment of a header is <= `MIN_ALIGN` and `aligned`
        // is aligned to at least `MIN_ALIGN` and has at least `MIN_ALIGN` bytes of padding before
        // it, it is safe to write a header directly before it.
        unsafe { core::ptr::write((aligned as *mut Header).offset(-1), Header(ptr)) };

        aligned
    } else {
        ptr
    }
}
```
**Note:** the `align_ptr` function assumes that the allocation size has been increased beforehand by `align_size`.

## Implement dealloc
The new `dealloc` looks like this:
```rust
unsafe fn dealloc(&self, ptr: *mut u8, layout: core::alloc::Layout) {
    let st = unsafe {
        match GLOBAL_SYSTEM_TABLE.get_mut() {
            Ok(x) => x,
            Err(_) => return,
        }
    };

    if layout.size() != 0 {
        let ptr = unsafe { unalign_ptr(ptr, layout.align()) };
        let r = memory_allocation_services::free_pool(st, ptr.cast());
        assert!(r.is_ok());
    }
}
```
The `unalign_ptr` function basically undoes what the `align_ptr` function did and gives back the original pointer.
```rust
#[inline]
unsafe fn unalign_ptr(ptr: *mut u8, align: usize) -> *mut u8 {
    if align > POOL_ALIGNMENT {
        // SAFETY: Because of the contract of `System`, `ptr` is guaranteed to be non-null
        // and have a header readable directly before it.
        unsafe { core::ptr::read((ptr as *mut Header).offset(-1)).0 }
    } else {
        ptr
    }
}
```

The same example as before should work even now without any change.

<br>

# Conclusion
With this, we now have a global allocator. I will soon be integrating it into the Rust std. In the next post, we will discuss implementing `stdin` for UEFI.

Consider [supporting me](@/pages/supportme.md) if you like my work.

<br>

# Helpful Links
- [uefi-spec-rs](https://github.com/Ayush1325/uefi-spec-rs)
