The point is to dream big.

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
