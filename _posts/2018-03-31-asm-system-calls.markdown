---
layout: post
title:  "System calls!"
date:   2018-03-31 01:04:48 +0530
categories: rust system-calls asm
---

For some random reason, I wanted to manually do system calls without any compiler doing that for me and started looking at ways to do that. I was able to do it in a couple of hours and this is what I ended up with. It is pretty barebones and not-so-ergonomic but hey, it's something!


```rust
#![feature(asm)]
#[cfg(unix)]

const O_WRONLY: i32 = 01;
const O_CREAT: i32 = 0100;
const S_IRWXU: i32 = 0700;

#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
fn open(filename: &[u8], flags: i32, mode: i32) -> i32 {
    let fd: i32;
    let filename_ptr = filename.as_ptr();
    unsafe {
        asm!("movl $$2, %eax;
              syscall"
             : "={rax}"(fd)
             : "{rdi}"(filename_ptr),
               "{rsi}"(flags)
               "{rdx}"(mode)
             : "memory"
             : "alignstack" "volatile");
    }
    return fd;
}

#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
fn write(fd: i32, buf: &str) -> i32 {
    let buflen = buf.len();
    let ptr = buf.as_ptr();
    let len: i32;
    unsafe {
        asm!("movl $$1, %eax;
             syscall"
             : "={rax}"(len)
             : "{rsi}"(ptr),
               "{edi}"(fd),
               "{rdx}"(buflen)
             : "memory"
             : "alignstack" "volatile");
    }
    return len;
}

fn main() {
    let filename = b"testme\0"; /* open expects a c string */
    let flags = O_WRONLY | O_CREAT;
    let fd = open(filename, flags, S_IRWXU);
    write(fd, "Hello world!");
}
```
This is an example of how to invoke the `open` and `write` system calls for an x86 linux machine. This is also the first time I am using inline assembly, unsafe blocks in rust.

`asm` is a rust macro which allows us to write assembly code in rust source code. It is similar to the `println` or `format` macros which we use to print or format strings. `asm` generates assembly code and allows us place variables in a particular register or set of registers, read the values of registers into a variable's memory location etc.

System calls are operating system specific ABI. They allow us to interact with the operating system. In the above example, I invoked two commonly used system calls: `open` and `write` used to do I/O. In our case, file I/O. Each system call is given a number and expects its arguments to be in certain registers specified in its ABI before being called.

For example, `open` expects a pointer to a string in the register `rdi` and the file flags in `rsi` and mode in `rdx`. Before returning, places its return value in `rax` which is a file descriptor. 

When you call the `open` function, the compiler takes care of placing the registers correctly for you and when `syscall` instruction is executed, control is given to the operating system. After the system call, the control is transferred back to the process. This allows processes to be run in `user` mode with tighter constraints on what they have access to and allows the operating system to execute with greater privileges - `kernel` mode(with a bit of help from the underlying hardware).

See [this](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/) for the full system call table for Linux.
See [LLVM inline assembly reference](http://llvm.org/docs/LangRef.html#inline-assembler-expressions).
For an extended example of how to use `asm`, see [this](https://stackoverflow.com/a/34484671).


Before you ask, yes of course there are bugs! The created file has weird permissions and I am too lazy to find out why :)

Thanks for reading this far! Hopefully, at least something makes sense.
