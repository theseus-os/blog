---
layout: post
title: "More progress porting wasmtime to Theseus"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## The <s>quest</s> port continues

This post covers the highlights of our ongoing work to port Wasmtime to Theseus; see the previous post(s) for more information.
Interested folks can [follow along with the ported Wasmtime code here](https://github.com/theseus-os/wasmtime/tree/theseus).

📢 Good news: `wasmtime-runtime` now builds on Theseus! 📢

### Completed initial port of `wasmtime-runtime`!
Last time we left off having finished several key features to support the needs of the `wasmtime-runtime` crate:
* Thread-Local Storage (TLS)
* Creation and management of Unix-like memory-mapped areas via the `region` and `libc` crates
  * Plus additions to `tlibc`, the Theseus-specific implementation of basic libc functions 
* Reading/writing of object files via the `object` crate
* Improvements to Theseus's `page_allocator` and `theseus_cargo` build tool to handle more complexity


The full list of required, target-agnostic dependencies is as follows:
```toml
[package]
name = "wasmtime-runtime"
...

[dependencies]
wasmtime-environ = { path = "../environ", version = "0.30.0" }  ## Ported to `no_std` previously
libc = { version = "0.2.82", default-features = false }         ## Ported to Theseus in prior post
region = "2.1.0"         ## Ported to Theseus in prior post
log = "0.4.8"            ## Supports `no_std`
memoffset = "0.6.0"      ## Supports `no_std`
indexmap = "1.0.2"       ## Supports `no_std`
thiserror = "1.0.4"      ## Use `thiserror_core2` instead
more-asserts = "0.2.1"   ## Ported to `no_std` in prior post
cfg-if = "1.0"           ## Supports `no_std`
backtrace = "0.3.61"     ## Ported to Theseus in this post
lazy_static = "1.3.0"    ## Supports `no_std`
rand = "0.8.3"           ## Offers `no_std`-compatible `SmallRng`
anyhow = "1.0.38"        ## Supports `no_std`, with code changes
```


Over the past several weeks, we have completed our modifications to `wasmtime-runtime` such that it now builds properly on Theseus!
The key missing parts were:

|   Crate / Feature    |        Summary        | Reason Needed for `wasmtime-runtime` |
|----------------------|-----------------------|--------------------------------------|
| `backtrace`          | Cross-platform crate for capturing stack traces    | For capturing and analyzing stack traces to see if any WASM module functions exist on the call stack |
| `std::path`          | Module for manipulating and parsing file paths | To refer to WASM module files, and for (de)serialization |
| `resume_unwind()`    | Continues a panic action (e.g., unwinding), but skips the registered panic hook | To continue propagating a panic across a native code-WASM boundary |
| `thiserror`/`anyhow` | Helper crates for convenient error handling | To derive the `Error` trait and easily return error types |
| Signal handling      | Registering signal handlers, e.g., for `SIGSEGV`, `SIGILL` | For catching OS-level exceptions that occur while executing native code compiled from WASM modules |


#### `Back`(`trace`) to the Future
The `wasmtime-runtime` crate uses `backtrace` to capture a stack trace when a *trap* occurs, such as a fault during WASM execution or another systems-level problem like Out Of Memory (OOM).
This trace is used to both:
1. Traverse the call stack to see if any stack frames from WASM code exist, and
2. Provide the user or caller of Wasmtime with more context about a runtime failure.

Porting `backtrace` was relatively simple; the primary changes required were to publicly expose more details about Theseus's custom unwinder.
Feel free to [check out the full changeset here](https://github.com/theseus-os/backtrace-rs/compare/5e15d73..c56bae0), summarized below:
* Theseus's unwinder calculates the register values for each stack frame. We simply use those values to provide `backtrace` with:
  * The stack frame's instruction pointer (i.e., *call site address*)
  * The current stack pointer at that execution point
  * The starting address of its containing function 
* Theseus supports *symbolication*: resolving an address into a symbol
  * This uses Theseus's crate management metadata, a [`CrateNamespace`](https://www.theseus-os.com/Theseus/doc/mod_mgmt/struct.CrateNamespace.html), which contains a map of all public symbols in that namespace
  * The key function is [`get_section_containing_address()`](https://www.theseus-os.com/Theseus/doc/mod_mgmt/struct.CrateNamespace.html#method.get_section_containing_address)
    * This is similar to the [`addr2line`](https://linux.die.net/man/1/addr2line) tool, but it works with Theseus's dynamically loading and linked code structure

The one remaining feature that our port of `backtrace` lacks is connecting a resolved symbol to its location: file path, line number, and column number.
This is conceptually easy to do but requires debug information to be parsed from an object file. 
Although [Theseus does support parsing DWARF debug info](https://www.theseus-os.com/Theseus/doc/debug_info/index.html), it isn't always available because debug info is typically stripped from object files to keep their size down.
Fortunately, the `backtrace` crate treats this information as optional, and thus Wasmtime doesn't require it to be available, so we can simply return `None` when asked for symbol location details.


#### The `Path` Forwards

Many Wasmtime crates use `std::path::{Path, PathBuf}` to refer to WASM module files that are JIT-compiled and loaded into a Wasmtime engine.
Thus, we must implement a version of path types that are API-compatible with Rust's `std::path` types in order to minimize the number of changes to Wasmtime itself.
You can find [the code for that here](https://github.com/theseus-os/Theseus/commit/6a4ba4f42ae407afe71dd77042fa15a523e15134), which is primarily a quick & dirty copy of the code from `std::path`.

Theseus already offers [its own `Path` type](https://www.theseus-os.com/Theseus/doc/path/struct.Path.html), which is similar but not identical to those in Rust `std::path`.
All we need is a simple glue code layer between `std::PathBuf` and Theseus's `path::Path`.

> One notable difference between `std::path` and `theseus_path` is that Theseus uses Rust `String` and `str` types natively, so there is no need for the equivalent types `OsString` and `OsStr`; these types become simple typedefs in Theseus:
> ```rust
> pub type OsString = String;
> pub type OsStr = str;
> ```


#### Relax and unwind
The `resume_unwind()` function is used in Wasmtime to carry on with the unwinding procedure after it has been *caught*. This is currently only used in the runtime's trap handling logic, which essentially continues unwinding after a trap that stemmed from a Rust-level panic, i.e., one deemed irrelevant to handling in-WASM traps.


Here is Theseus's [implementation of `resume_unwind()` with code that tests it](https://github.com/theseus-os/Theseus/commit/ca960df2a61d807f23514469a94355ee2689c556#diff-d17ff1523da38049c7e75df2747fc33e3a18dea8c6a57ecb1ba5590ff6f1a699).
Although this is conceptually tricky, the implementation is quite straightforward — simply start unwinding from the current point without starting from the regular panic handler.
You can test this in Theseus by invoking the test program `unwind_test -c`.

To actually use our new `resume_unwind()` function in `wasmtime-runtime`, we add [the following very simple code block](https://github.com/theseus-os/wasmtime/blob/076a39724c76ad29423db5e4aae7ef1d15530693/crates/runtime/src/traphandlers.rs#L267-L272).
```rust
#[cfg(feature = "std")]
std::panic::resume_unwind(panic);
#[cfg(target_os = "theseus")]
theseus_catch_unwind::resume_unwind(
    KillReason::Panic(PanicInfoOwned::from_payload(panic))
);
```

> The notable difference between Theseus's `resume_unwind` and Rust's `std::panic::resume_unwind` is that we allow unwinding to occur from both a language-level panic and *beneath* the language level after a CPU-level "machine" exception. 
> Thus, Theseus's panic payloads expect a [`KillReason`]() rather than an erased type `Box<Any>`, so we must handle that minor difference.


#### Oh, an `Error` Occurred? <s>Anyway</s> Anyhow...
In yet another tribute to D. Tolnay, Wasmtime uses his superb `anyhow` and `thiserror` crates for convenient error handling across nearly every source file and function.
While `anyhow` technically supports `no_std` environments like Theseus, it cannot accommodate the same API.

> Thus, we must change ***every*. *single*. *usage*.** of `anyhow` in the whole Wasmtime code base.

The majority of the changes simply require use to add this snippet to any `Result` type before returning or unwrapping it:
```rust
.map_err(anyhow::Error::msg)?  // convert the `Err` into `anyhow::Error`
```
because in `no_std` environments, `anyhow::Error` does not `impl std::error::Error` and thus cannot perform the implicit conversion; we must do it explicitly.
This rough edge is currently unavoidable, as evidenced by [`anyhow`'s documentation](https://github.com/dtolnay/anyhow#no-std-support).

As you can see, most of these changes are functionally unnecessary. The real reason for such tedium is that the `Error` trait is defined in Rust's `std` library and is thus unavailable for use in `no_std` environments that only use `core` or `alloc`.

Thankfully, supporting `thiserror` is a bit easier, thanks to the [`thiserror_core2` crate](https://github.com/bbqsrc/thiserror-core2) that ports it to emit a derivation of the [`core2::error::Error` trait](https://github.com/technocreatives/core2). We can simply use this as a drop-in replacement for `thiserror` because we already use `core2::error::Error` as a substitute for `std::error::Error`.

💤💭💤 We dream of the day when [@Jane Lusby](https://github.com/yaahc)'s excellent work on moving the `Error` trait into `core` is completed! Once that lands, we won't need to bother with all this pablum.


#### The Last <s>Jedi</s> Dependency: Signal Handling

The last remaining feature needed to finish porting `wasmtime-runtime` is signal handling.
This is needed for Wasmtime to be able to catch traps that occur when executing WASM code that was JIT-compiled into native code, among other purposes.

Theseus doesn't offer POSIX-like signals because they're unsafe and unnecessary in a safe-language OS, but it does implement handlers for [CPU exceptions](https://wiki.osdev.org/Exceptions), e.g., page faults, general protection faults, etc.
However, we previously did not allow third-party crates to register handlers (callbacks) 

Registering  "signal" (CPU exception) handlers is now supported in Theseus as of [commit ](https://github.com/theseus-os/Theseus/commit/09ccbfc644da93e98ef11c9532ef926f1ee7b4f4).
The initial implementation is limited to the following four categories of "signals", which was loosely based on the set of signals that `wasmtime-runtime` cares about.
```rust
/// The possible "signals" that may occur due to CPU exceptions.
pub enum Exception {
    /// (SIGSEGV) Bad virtual address, unexpected page fault.
    InvalidAddress     = 0,
    /// (SIGILL) Invalid opcode, malformed instruction, etc.
    IllegalInstruction = 1,
    /// (SIGBUS) Bad memory alignment, non-existent physical address.
    BusError           = 2,
    /// (SIGFPE) Bad arithmetic operation, e.g., divide by zero.
    ArithmeticError    = 3,
}
```

Each task can register one handler function for each category of exceptions:
```rust 
pub trait ExceptionHandler = FnOnce(&ExceptionContext) -> Result<(), ()>;

pub fn register_handler(
    exception: Exception,
    handler: Box<dyn ExceptionHandler>,
) -> Result<(), ()> {
    HANDLERS.with(|handlers| {
        let handler_slot = &handlers[exception as usize];
        if handler_slot.borrow().is_some() {
            return Err(());
        }
        *handler_slot.borrow_mut() = Some(handler);
        Ok(())
    })
}
```

and the Thread-Local Storage (TLS) areas are used to efficiently access the handlers registered for each given task:

```rust
thread_local!{
    /// The "signal" handlers registered for each task.
    static HANDLERS: [RefCell<Option<Box<dyn ExceptionHandler>>>; 4] = Default::default();
}
```

When an exception occurs, the CPU jumps synchronously to the kernel function specified in the interrupt descriptor table (IDT).
We simply [add a condition to each IDT exception function](https://github.com/theseus-os/Theseus/blob/13406ed975d54b5d72afb313df370d0a4477ce01/kernel/exceptions_full/src/lib.rs#L185-L200) to check for and obtain the relevant registered exception handler (if one exists).
The registered handler is then invoked with the following contextual information about said exception; the key information is the `instruction_pointer`.
```rust
/// Information that is passed to a registered [`ExceptionHandler`]
/// about an exception that occurred during execution.
pub struct ExceptionContext {
    pub instruction_pointer: VirtualAddress,
    pub stack_pointer: VirtualAddress,
    pub exception: Exception,
    pub error_code: Option<ErrorCode>,
}
```

This feature is [used in the trap handling component](https://github.com/theseus-os/wasmtime/blob/e333d4441ad3bf634476e5b34e79fc3ff98f4fdb/crates/runtime/src/traphandlers/theseus.rs#L40-L48) of our port of `wasmtime-runtime`, which needs the above context to determine whether the exception occurred in a section of code that came from a compiled WASM module.
Of course, the handlers must first be registered as part of the [platform-specific init procedure shown here](https://github.com/theseus-os/wasmtime/blob/e333d4441ad3bf634476e5b34e79fc3ff98f4fdb/crates/runtime/src/traphandlers/theseus.rs#L16-L26).

## Onwards and Upwards

With that, our port of `wasmtime-runtime` is complete!
We're nearly done with the "minimum viable port" of wasmtime functionality, as the only remaining crates are `wasmtime-jit` and the top-level `wasmtime` crate itself.
Once those are complete, we will publish a longer-form post about the journey to get wasmtime running on Theseus.

Be on the lookout for such good news soon!

## Miscellaneous Contributions
* Fixed an [issue with TLS sections](https://github.com/theseus-os/Theseus/commit/78d0e008f9b5b89a11fca388442f395cde040569), which required adding support for loading `.tbss` (TLS BSS) sections 
* [@Jacob Earle](https://github.com/jacob-earle) implemented a [rate-monotonic scheduler](https://github.com/theseus-os/Theseus/commit/23bcfce0eb4303cb21ea3ff38afba8ae04013eab) for pseudo-real-time task scheduling
  * Adds a [`sleep` crate](https://www.theseus-os.com/Theseus/doc/sleep/index.html) that handles blocking a task for a given number of system ticks
  * Supports the notion of "periodic" tasks that run a short workload, go to sleep, and then wake back up at the beginning of every period
* [Updated to the latest Rust 1.61 nightly version](https://github.com/theseus-os/Theseus/commit/be9e9ed2c4a4baef45e321b2aa1fe50ea399218d)
  * This was required to fix the ABI issues in LLVM, wherein the `"x86-interrupt"` ABI for exception handlers didn't properly match what the CPU pushes onto the stack before jumping to said handler
  * Needed as part of [the aforementioned effort](#the-last-sjedis-dependency-signal-handling) to support registering third-party "signal" (exception) handlers in Theseus
* Kevin contributed a [tiny PR](https://github.com/rust-osdev/x86_64/pull/354) back to the `x86_64` crate
  * We had been using our own outdated fork for a while, but we now use the [latest version of the `x86_64` crate](https://github.com/theseus-os/Theseus/commit/7d979328b3a6c2b59a29aa0b43d96f5997b36171) with a tiny addition to support a `LockedIDT` type.
* [@Ramla Ijaz](https://github.com/Ramla-I) finished implementing [packet tranmission for the Mellanox 100GiB NIC](https://github.com/theseus-os/Theseus/commit/700daa0f959243c04e424c64649ca5660b5cb210)
