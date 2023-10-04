The point is to dream big.

Canton is my pet project, where I aim to build a simple chip, write an OS on it, write a C compiler for this target, and then write a simple shell and core utils in C. Ideally, I should be able to connect to internet via TLS1.3, and then write a simple web browser in Qt. Some things along the way: a linker, assembler, and re-implementing ELF would be nice. Currently I am using neovim with a few plugins, but I'd also want to rewrite `vi` and use that. 

- [ ] Rust Operating system (As a simple practice)
- [ ] OS in C from scratch
- [ ] Tiny CPU in VeriLog
- [ ] Simple Core utils in C
- [ ] Simple Shell in C
- [ ] Self-compiling compiler in C (very minimal)
- [ ] Network stack in C (allows to send and receive packets)

# Writing an OS in Rust Series Notes

Kernel-level programming often avoids using the standard library (such as C standard library) because it relies on low-level operations and lacks the infrastructure and safety features provided by the standard library, which can introduce unpredictability and potential security risks in kernel code.

## Creating a Free-standing Rust Binary

First thing we do, is create a [free-standing Rust binary](https://os.phil-opp.com/freestanding-rust-binary/). We need to do this because we can't depend on any OS features (as explained above), but we will still use the core features of Rust.

```rust
#![no_std]
```

This line disables the implicit linking of standard libraries. Notice now `println` won't work. If you've ever written a kernel module before, this might be a throwback to the fact that `println` in C doesn't work at the kernel level when you do not import `stdlib.h`. The only difference here is we explicitly delink the "stdlib" in Rust because for whatever reason, Rust by default links it implicitly.

## Handling Panics

[Panic](https://doc.rust-lang.org/stable/book/ch09-01-unrecoverable-errors-with-panic.html) is an important artifact in Rust that ships with the standard library. For us, we will simply define our own panic function. Import `core::panic::PanicInfo` ([PanicInfo Documentation](https://doc.rust-lang.org/core/panic/struct.PanicInfo.html)) and define the behavior by using the `#[panic_handler]` procedural macro. We'll just throw it into a loop for now (Note: Since this function should never return, we mark it as `!` return type).

## Disabling Stack Unwinding

`eh_personality` is usually a funny descriptor of people from Delhi, but here it is used for marking the functions for stack unwinding (cleaning up the stack frames and such in case of panic exits, etc.). This ensures we have a nice, clean, and free memory once we have an abnormal exit. However, it's complex and sometimes needs OS-specific libraries.

So we will simply disable unwinding, and abort on any panic instead from the `cargo.toml`.

## Setting the Entry Point

When you try compiling now, we will be missing the `start` macro, which is the entry point.

`main` functions are not the first functions that run in Rust. In a typical Rust binary, `crt0` is a C runtime library that sets up the environment. This includes creating a stack and placing the arguments on it.

This is not unusual; any amount of `gdb` debugging will have you know the whole bits and registers at the beginning. If you're someone that learned `gdb` and C, you might wonder about the runtime systems of C itself. [More information here](https://stackoverflow.com/questions/42728239/runtime-system-in-c).

The C runtime from `crt0` then calls the Rust runtime, which is pretty straightforward - it just sets up Stack Guards and such.

We don't have any of this, so we disable `main` by doing `#![no_main]`. We have no underlying runtime that calls our `main`, so we just remove it.

Now let's overwrite the OS entry point with our own `_start` function:

```rust
#[no_mangle] // disable name mangling
pub extern "C" fn _start() -> ! {
    loop {}
}
```

Name mangling is great in usual compilers because it allows us to have different method signatures with the same method names but different parameters. However, we unambiguously want access to the same `_start` function.


Creating a minimal Rust kernel 
ref. https://os.phil-opp.com/minimal-rust-kernel/

The aim is to create a minimal 64-bit Rust kernel for the x86 architecture. 

When you turn on a computer, a firmware code stored in the motherboard ROM starts a process called POST, which detects current RAM, pre-initializes CPU and hardware. It then looks for a bootable disk.

There are two standards for this: BIOS (Basic I/O system) and UEFI (unified extensible firmware interface). 
BIOS is very widely compatible, but this has its problems.
CPU is put into a 16 bit compatibility mode called real mode. 
When the BIOS finds a bootable disk, the control is transferred to a bootloader (which is a 512 byte portion of exec code found at the disk’s beginning).
The second stage of the bootloader is loaded by the first stage.
The bootloader has to determine the location of the kernel image on the disk and load it into the memory
It also needs to switch the CPU from 16 bit real mode, to 32 bit protected mode, and 64 bit long mode, where 64 bit registers (eax → rax if you recall) are available. 
It’s last job is to query certain information from the BIOS and pass it to the OS kernel (such as a memory map)
For now, using bootimage tool. (I will be implementing my own linker, assembler, simple BIOS and bootloader soon).
To make a Kernel mulitboot compilant (it’s a standard), one just needs to insert a so-called multiboot header at the beginning of the kernel file. This makes it easier to boot an OS from GRUB, however there are problems too
32 bit protected mode only supported, which means you will have to config CPU to switch to 64 bit
They are designed to make the bootloader simple instead of the kernel. For example, the kernel needs to be linked with an adjusted default page size, because GRUB can’t find the Multiboot header otherwise. Another example is that the boot information, which is passed to the kernel, contains lots of architecture-dependent structures instead of providing clean abstractions.
Both GRUB and the Multiboot standard are only sparsely documented.
GRUB needs to be installed on the host system to create a bootable disk image from the kernel file. This makes development on Windows or Mac more difficult.
target-triple allow cargo to know the CPU architecture. We don’t have an underlying OS, so we will define our OS to be none (which rust allows for).
Further, add Rust’s LDD linker instead of using the platform’s default linker. 
Set panic-strategy to abort since we have disabled stack unwinding.
Disable the red zone, which is an optimisation of the System V ABI that allows functions to temporarily use 128 bytes below the stack frame without adjusting the pointer.
https://forum.osdev.org/viewtopic.php?t=21720 for reference of sample errors
Claude summary: After spending six days debugging a critical issue in a hobby x86-64 kernel, it was discovered that frequent interrupts from the monotonic PIT timer were corrupting the kernel state. Initially, it was suspected that the interrupt handler code was responsible, but even the simplest handler code didn't resolve the issue. After extensive disassembly and analysis, it was found that GCC was generating assembly code that used the red zone, a 128-byte area below the stack, which was interrupt-unsafe according to the x86-64 ABI. This led to the corruption of the kernel state during interrupts. The solution was as simple as instructing GCC not to use the red zone with the -mno-red-zone flag. This fix resolved the bug, and the kernel performed reliably even under heavy interrupt loads. The debugging process was aided by using the Bochs binary single-stepping debugger, and special thanks were given to Brendan for their valuable advice throughout the troubleshooting process.
SIMD works by breaking down a computational task into smaller, identical operations and then applying these operations simultaneously to multiple data elements. A single instruction, as the name suggests, guides these parallel operations, ensuring consistency and efficiency. This technique is particularly effective for tasks where the same operation is performed on a large dataset, such as image and video processing or numerical simulations, as it significantly accelerates processing speed and enhances overall performance.

However, this also means in OS where interrupts are frequent, we will be throttled by this. So we will disable this.

```
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
    "linker-flavor": "ld.lld",
    "linker": "rust-lld",
    "panic-strategy": "abort",
    "disable-redzone": true,
    "features": "-mmx,-sse,+soft-float"
}
```

mmx and sse are used for SIMD support, which we don’t need, so we do -mmx,-sse and then we add in +soft-float because soft-float emulates all floating point operations through software functions based on normal integers. This is great because x86_64 requires SIMD registers by default, we will just circumvent the problem.

When we try to build now, we will get a `rustc --explain E0463` error. This is not good, because we need the basic Rust types such as Result, Option etc. This is (again, for whatever reason) implicitly linked in. The core library is distributed as a precompiled library, so we get valid support for valid hosts. We need to recompile core for ourselves.

To do this, we need to actually edit the ` config.toml ` for cargo itself. Make a .cargo add in a config. Now, get the rust source code with ` rustup component add rust-src `.

Further Notes
Also very interesting: https://www.davidsalomon.name/assem.advertis/asl.pdf

