+++
title = "Integration Tests"
order = 5
path = "integration-tests"
date  = 2018-05-18
template = "second-edition/page.html"
+++

In this post we complete the testing picture by implementing a basic integration test framework, which allows us to run tests on the target system. The idea is to run tests inside QEMU and report the results back to the host through the serial port.

<!-- more -->

This blog is openly developed on [Github]. If you have any problems or questions, please open an issue there. You can also leave comments [at the bottom].

[Github]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments

## Overview

In the previous post we added support for unit tests. The goal of unit tests is to test small components in isolation to ensure that all of them work as intended. The tests are run on the host machine and thus shouldn't rely on architecture specific functionality.

To test the interaction of the components, both with each other and the system environment, we can write _integration tests_. Compared to unit tests, ìntegration tests are more complex, because they need to run in a realistic environment. What this means depends on the application type. For example, for webserver applications it often means to set up a database instance. For an operating system kernel like ours, it means that we run the tests on the target hardware without an underlying operating system.

Running on the target architecture allows us to test all hardware specific code such as the VGA buffer or the effects of [page table] modifications. It also allows us to verify that our kernel boots without problems and that no [CPU exception] occurs.

[page table]: https://en.wikipedia.org/wiki/Page_table
[CPU exception]: https://wiki.osdev.org/Exceptions

In this post we will implement a very basic test framework that runs integration tests inside instances of the [QEMU] virtual machine. It is not as realistic as running them on real hardware, but it is much simpler and should be suffient as long as we only use standard hardware that is well supported in QEMU.

[QEMU]: https://www.qemu.org/

## The Serial Port

The naive way of doing an integration test would be to add some assertions in the code, launch QEMU, and manually check if a panic occured or not. This is very cumbersome and not practical if we have hundreds of integration tests. So we want an automated solution that runs all tests and fails if not all of them pass.

Such an automated test framework needs to know whether a test succeeded or failed. It can't look at the screen output of QEMU, so we need a different way of retrieving the test results on the host system. A simple way to achieve this is by using the [serial port], an old interface standard which is no longer found in modern computers. It is easy to program and QEMU can redirect the bytes sent over serial to the host's standard output or a file.

[serial port]: https://en.wikipedia.org/wiki/Serial_port

The chips implementing a serial interface are called [UARTs]. There are [lots of UART models] on x86, but fortunately the only differences between them are some advanced features we don't need. The common UARTs today are all compatible to the [16550 UART], so we will use that model for our testing framework.

[UARTs]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter
[lots of UART models]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter#UART_models
[16550 UART]: https://en.wikipedia.org/wiki/16550_UART

### Port I/O
 TODO

### Implementation

There is already the [`uart_16550`] crate, so we don't need to write our own driver for it. To add it as a dependency, we update our `Cargo.toml` and `main.rs`:

[`uart_16550`]: https://docs.rs/uart_16550

```toml
# in Cargo.toml

[dependencies]
uart_16550 = "0.1.0"
```

```rust
// in src/main.rs

extern crate uart_16550;
```

The `uart_16550` crate contains a `SerialPort` struct that represents the UART registers, but we still need to construct an instance of it ourselves. For that we create a new `serial` module with the following content:

```rust
// in src/main.rs

mod serial
```

```rust
// in src/serial.rs

use uart_16550::SerialPort;
use spin::Mutex;

lazy_static! {
    pub static ref SERIAL1: Mutex<SerialPort> = {
        let mut serial_port = SerialPort::new(0x3F8);
        serial_port.init();
        Mutex::new(serial_port)
    };
}
```

Like with the [VGA text buffer], we use a spinlock and `lazy_static` to create a `static`. However, this time we use `lazy_static` to ensure that the `init` method is called before first use. We're using the port address `0x3F8`, which is standard for the first serial interface.

[VGA text buffer]: ./second-edition/posts/03-vga-text-buffer/index.md#spinlocks

To make the serial port easily usable, we add `serial_print!` and `serial_println!` macros:

```rust
pub fn print(args: ::core::fmt::Arguments) {
    use core::fmt::Write;
    let _ = COM1.lock().write_fmt(args);
}

/// Prints to the host through the serial interface.
macro_rules! serial_print {
    ($($arg:tt)*) => {
        $crate::serial::print(format_args!($($arg)*));
    };
}

/// Prints to the host through the serial interface, appending a newline.
macro_rules! serial_println {
    () => (serial_print!("\n"));
    ($fmt:expr) => (serial_print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => (serial_print!(concat!($fmt, "\n"), $($arg)*));
}
```

The `SerialPort` type already implements the [`fmt::Write`] trait, so we don't need to provide an implementation.

[`fmt::Write`]: https://doc.rust-lang.org/nightly/core/fmt/trait.Write.html

Now we can print to the serial interface in our `main.rs`:

```rust
// in src/main.rs

#[macro_use]
mod serial;

#[cfg(not(test))]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!"); // prints to vga buffer
    hprintln!("Hello Host{}", "!"); // prints to serial

    loop {}
}
```

Note that we need to add the `#[macro_use]` attribute to the `mod serial` declaration, because otherwise the `hprintln` macro is not imported.

### QEMU Arguments

To see the serial output in QEMU, we can use the `-serial` argument to redirect the output to stdout:

```
> qemu-system-x86_64 \
    -drive format=raw,file=target/x86_64-blog_os/debug/bootimage-blog_os.bin \
    -serial mon:stdio
warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]
Hello Host!
```

If you chose a different name than `blog_os`, you need to update the paths of course. Note that you can no longer exit QEMU through `Ctrl+c`. As an alternative you can use `Ctrl+a` and then `x`.

As an alternative to this long command, we can pass the argument to `bootimage run`, with an additional `--` to separate the build arguments (passed to cargo) from the run arguments (passed to QEMU).

```
bootimage run -- -serial mon:stdio
```

Instead of standard output, QEMU supports [many more target devices][QEMU -serial]. For redirecting the output to a file, the argument is:

[QEMU -serial]: https://qemu.weilnetz.de/doc/qemu-doc.html#Debug_002fExpert-options

```
-serial file:output-file.txt
```

## Shutting Down QEMU

Right now we have an endless loop at the end of our `_start` function and need to close QEMU manually. This does not work for automated tests. We could try to kill QEMU automatically from the host, for example after some special output was sent over serial, but this would be a bit hacky and difficult to get right. The cleaner solution would be to implement a way to shutdown our OS. Unfortunatly this is relatively complex, because it requires implementing support for either the [APM] or [ACPI] power management standard.

[APM]: https://wiki.osdev.org/APM
[ACPI]: https://wiki.osdev.org/ACPI

Luckily, there is an escape hatch: QEMU supports a special `isa-debug-exit` device, which provides an easy way to exit QEMU from the guest system. To enable it, we add the following argument to our QEMU command:

```
-device isa-debug-exit,iobase=0xf4,iosize=0x04
```

The `iobase` specifies on which port address the device should live and the `iosize` specifies the port size (`0x04` means four bytes). Now the guest can write a value to the `0xf4` port and QEMU will exit with [exit status] `(passed_value << 1) | 1`.

[exit status]: https://en.wikipedia.org/wiki/Exit_status

To write to the I/O port, we use the [`x86_64`] crate:

[`x86_64`]: https://docs.rs/x86_64

```toml
# in Cargo.toml

[dependencies]
x86_64 = "0.2.0"
```

```rust
// in src/main.rs

extern crate x86_64;

pub unsafe fn exit_qemu() {
    use x86_64::instructions::port::Port;

    let mut port = Port::<u32>::new(0xf4);
    port.write(0);
}
```

We mark the function as `unsafe` because it relies on the fact that a special QEMU device is attached to the I/O port with address `0xf4`. For the port type we choose `u32` because the `iosize` is 4 bytes. As value we write a zero, which causes QEMU to exit with exit status `(0 << 1) | 1 = 1`.

Note that we could also use the exit status instead of the serial interface for sending the test results, for example `1` for success and `2` for failure. However, this wouldn't allow us to send panic messages like the serial interface does and would also prevent us from replacing `exit_qemu` with a proper shutdown someday. Therefore we continue to use the serial interface and just always write a `0` to the port.

We can now test the QEMU shutdown by calling `exit_qemu` from our `_start` function:

```rust
#[cfg(not(test))]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!"); // prints to vga buffer
    println!("Hello Host{}", "!"); // prints to serial

    unsafe { exit_qemu(); }

    loop {}
}
```

You should see that QEMU immediately closes after booting.

## Hiding QEMU

We are now able to launch a QEMU instance that writes its output to the serial port and automatically exits itself when it's done. So we no longer need the VGA buffer output or the graphical representation that still pops up. We can disable it by passing the `-display none` parameter to QEMU. The full command looks like this:

```
qemu-system-x86_64 \
    -drive format=raw,file=target/x86_64-blog_os/debug/bootimage-blog_os.bin \
    -serial mon:stdio \
    -device isa-debug-exit,iobase=0xf4,iosize=0x04 \
    -display none
```

Or, with `bootimage run`:

```
bootimage run -- \
    -serial mon:stdio \
    -device isa-debug-exit,iobase=0xf4,iosize=0x04 \
    -display none
```

Now QEMU runs completely in the background and no window is opened anymore. This is not only less annoying, but also allows our test framework to run in environments without a graphical user interface, such as [Travis CI].

[Travis CI]: https://travis-ci.com/

## Test Organization

Right now we're doing the serial output and the QEMU exit from the `_start` function in our `main.rs` and can no longer run our kernel in a normal way. We could try to fix this by adding an `integration-test` [cargo feature] and using [conditional compilation]:

[cargo feature]: https://doc.rust-lang.org/cargo/reference/manifest.html#the-features-section
[conditional compilation]: https://doc.rust-lang.org/reference/attributes.html#conditional-compilation

```toml
# in Cargo.toml

[features]
integration-test = []
```

```rust
// in src/main.rs

#[cfg(not(feature = "integration-test"))] // new
#[cfg(not(test))]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!"); // prints to vga buffer

    // normal execution

    loop {}
}

#[cfg(feature = "integration-test")] // new
#[cfg(not(test))]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello Host{}", "!"); // prints to serial

    run_test_1();
    run_test_2();
    // run more tests

    unsafe { exit_qemu(); }

    loop {}
}
```

However, this approach has a big problem: All tests run in the same kernel instance, which means that they can influence each other. For example, if `run_test_1` misconfigures the system by loading an invalid [page table], it can cause `run_test_2` to fail. This isn't something that we want because it makes it very difficult to find the actual cause of an error.

[page table]: https://en.wikipedia.org/wiki/Page_table

Instead, we want our test instances to be as independent as possible. If a test wants to destroy most of the system configuration to ensure that some property still holds in catastrophic situations, it should be able to do so without needing to restore a correct system state afterwards. This means that we need to launch a separate QEMU instance for each test.

With the above conditional compilation we only have two modes: Run the kernel normally or execute _all_ integration tests. To run each test in isolation we would need a separate cargo feature for each test with that approach, which would result in very complex conditional compilation bounds and confusing code.

A better solution is to create an additional executable for each test.

### Additional Test Executables

Cargo allows to add [additional executables] to a project by putting them inside `src/bin`. This allows us to create a separate executable for each unit tests. For example, let's create the skeleton for a `test-something` executable:

[additional executables]: https://doc.rust-lang.org/cargo/reference/manifest.html#the-project-layout

```rust
// in src/bin/test-something.rs

#![feature(lang_items)] // required for defining the panic handler
#![no_std] // don't link the Rust standard library
#![cfg_attr(not(test), no_main)] // disable all Rust-level entry points

/// This function is the entry point, since the linker looks for a function
/// named `_start_` by default.
#[cfg(not(test))]
#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // TODO run tests
    loop {}
}

/// This function is called on panic.
#[cfg(not(test))]
#[lang = "panic_fmt"]
#[no_mangle]
pub extern "C" fn rust_begin_panic(
    _msg: core::fmt::Arguments,
    _file: &'static str,
    _line: u32,
    _column: u32,
) -> ! {
    loop {}
}
```

By providing a new implementation for `_start` we can create a minimal test case that only tests one specific thing and is independent of the rest. For example, if we don't print anything to the VGA buffer, the test still succeeds even if the `vga_buffer` module is broken.

We can now run this executable in QEMU by passing a `--bin` argument to `bootimage`:

```
bootimage run --bin test-something
```

It should build the `test-something.rs` executable instead of `main.rs` and launch an empty QEMU window (since we don't print anything). So this approach allows us to create completely independent executables without cargo features or conditional compilation, and without cluttering our `main.rs`.

However, there is a problem: This is a completely separate executable, which means that we can't access any functions from our `main.rs`, including `hprintln`, `exit_qemu`, and the `serial` module. Duplicating the code would work, but we would also need to copy everything we want to test. This would mean that we no longer test the original function but only a possibly outdated copy.

Fortunately there is a way to share most of the code between our `main.rs` and the testing binaries: We move most of the code from our `main.rs` to a library that we can include from all executables.

### Split Off A Library

Cargo supports hybrid projects that are both a library and a binary. We only need to create a `src/lib.rs` file and split the contents of our `main.rs` in the following way:

```rust
// src/lib.rs

#![feature(const_fn)]
#![no_std] // don't link the Rust standard library

extern crate spin;
extern crate volatile;
#[macro_use]
extern crate lazy_static;
extern crate uart_16550;
extern crate x86_64;

#[cfg(test)]
extern crate array_init;
#[cfg(test)]
extern crate std;

// NEW: We need to add `pub` here to make them accessible from the outside
pub mod vga_buffer;
pub mod serial;

pub unsafe fn exit_qemu() {
    use x86_64::instructions::port::Port;

    let mut port = Port::<u32>::new(0xf4);
    port.write(0);
}
```

```rust
// src/main.rs

#![feature(lang_items)] // required for defining the panic handler
#![no_std] // don't link the Rust standard library
#![cfg_attr(not(test), no_main)] // disable all Rust-level entry points
#![cfg_attr(test, allow(dead_code, unused_macros))] // allow unused code in test mode

// NEW: Add the library as dependency (same crate name as executable)
#[macro_use]
extern crate blog_os;

/// This function is the entry point, since the linker looks for a function
/// named `_start_` by default.
#[cfg(not(test))]
#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!");

    loop {}
}

/// This function is called on panic.
#[cfg(not(test))]
#[lang = "panic_fmt"]
#[no_mangle]
pub extern "C" fn rust_begin_panic(
    _msg: core::fmt::Arguments,
    _file: &'static str,
    _line: u32,
    _column: u32,
) -> ! {
    loop {}
}
```

So we move everything except `_start` and `rust_begin_panic` to `lib.rs`, make the `vga_buffer` and `serial` modules public, and add an `extern crate` definition to our `main.rs`.

This doesn't compile yet, because Rust's macros are not exported over crate boundaries by default. To export our printing macros, we need to add the `#[macro_export]` attribute to them:

```rust
// in src/vga_buffer.rs

#[macro_export]
macro_rules! print {…}

#[macro_export]
macro_rules! println {…}

// in src/serial.rs

#[macro_export]
macro_rules! hprint {…}

#[macro_export]
macro_rules! hprintln {…}
```

Now everything should work exactly as before, including `bootimage run` and `cargo test`.

### Test Basic Boot

We are finally able to create our first integration test executable. We start simple and only test that the basic boot sequence works and the `_start` function is called:

```rust
// in src/bin/test-basic-boot.rs

#![feature(lang_items)] // required for defining the panic handler
#![feature(const_fn)]
#![no_std] // don't link the Rust standard library
#![cfg_attr(not(test), no_main)] // disable all Rust-level entry points

#[macro_use]
extern crate blog_os;

use blog_os::exit_qemu;

/// This function is the entry point, since the linker looks for a function
/// named `_start_` by default.
#[cfg(not(test))]
#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    hprintln!("ok");

    unsafe { exit_qemu(); }
    loop {}
}

/// This function is called on panic.
#[cfg(not(test))]
#[lang = "panic_fmt"]
#[no_mangle]
pub extern "C" fn rust_begin_panic(
    msg: core::fmt::Arguments,
    file: &'static str,
    line: u32,
    column: u32,
) -> ! {
    hprintln!("failed");

    hprintln!("panic: {} at {}:{}:{}", msg, file, line, column);

    unsafe { exit_qemu(); }
    loop {}
}
```

We don't do something special here, we just print `ok` if `_start` is called and `failed` with the panic message when a panic occurs. Let's try it:

```
> bootimage run --bin test-basic-boot -- \
    -serial mon:stdio -display none \
    -device isa-debug-exit,iobase=0xf4,iosize=0x04
Building kernel
   Compiling blog_os v0.2.0 (file:///…/blog_os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.19s
    Updating registry `https://github.com/rust-lang/crates.io-index`
Creating disk image at target/x86_64-blog_os/debug/bootimage-test-basic-boot.bin
warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]
ok
```

We got our `ok`, so it worked! Try inserting a `panic!()` before the `ok` printing, you should see output like this:

```
failed
panic: explicit panic at src/bin/test-basic-boot.rs:16:5
```

But what happens when we remove all printing and just `loop` endlessly? The QEMU process doesn't exit and keeps running forever. Our test runner must handle this case somehow, for example using a timeout.

(If you try the endless loop, remember that you can exit QEMU via `Ctrl+a` and then `x`.)

## Bootimage Test

TODO

- uses cargo metadata to find test-* binaries
- compiles and executes them, redirects output to file
- checks file for `ok`
- prints results

## Summary

TODO

TODO update date

## What's next?
In the next post, we will explore _CPU exceptions_. These exceptions are thrown by the CPU when something illegal happens, such as a division by zero or an access to an unmapped memory page (a so-called “page fault”). Being able to catch and examine these exceptions is very important for debugging future errors. Exception handling is also very similar to the handling of hardware interrupts, which is required for keyboard support.
