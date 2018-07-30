# The Signal Hook

As promised in the [previous article](2018-06-24-arc-more-atomic.md) (thanks for
all the valuable feedback ‒ I didn't have the time to act on it yet, but I
will), this talks about Unix signal handling.

Long story short, I wasn't happy about the signal handling story in Rust and
this is my attempt at improving it with the [signal-hook] library.

## What is a signal?

If you come from the Windows world or didn't have the opportunity to play with
signals more closely, this is a short intro. I apologise for some omissions, but
describing it fully would lead to a complete book, not one article ‒ just think
every second sentence ends with „unless you configured it to do it something
slightly different“ or „with the exception of this special situation, when it
actually does something else“.

Every process can receive so called signals. When a signal is received, a
dedicated piece of code is run inside the process. This is an ancient messaging
system and allows some really cool uses. But as it is said, Unix gives you
enough rope to hang yourself and it is doubly true with signals.

When a signal is delivered to a process (sent by other process or as a
consequence of some action, like executing an illegal instruction), the kernel
takes a list of threads that didn't mask the signal (didn't state they are not
interested in it right now), picks one of them, interrupts it and places the
designated function on top of that thread's stack. It lets the function run and
once it finishes, it resumes the original thread code.

If there are no such threads at the moment, the signal becomes pending. It is
either delivered once the situation changes, or the application can actively
„pull“ signals from the queue by various means.

## Problems of signals

There are several problems with signals. Some of them are general over
languages, some are specific to Rust.

### Signals are global unique resource

There's only one signal handler for each signal number per the whole program. If
a new one is installed, the old one is replaced by it. This is somewhat OK if
you are the top-level application, but if you're a library, you want to be able
to share the same process with other libraries. The call to install a signal
handler returns the old one, so you can chain the functions manually, but not
everyone does that and it is tedious and
[hard to do correctly](https://github.com/vorner/signal-hook/blob/fecac5f56e191d80ce2416d0d6324377bfb998fc/src/lib.rs#L233)

If using some means to pull pending signals actively out of the kernel, the
first library to come gets the signal and others are out of luck too.

### What you can do inside a signal handler is severely limited

A signal handler can happen to you at an arbitrarily bad time. But it runs
inside your own thread and your thread won't continue until it terminates. This
is worse than switching threads. Do you hold a mutex when the signal handler
runs, one that the signal handler would like to lock too? That's a cool way to
deadlock (or worse). Do you have some data structure in an inconsistent state?
See what the signal handler finds in there. And, by the way, the memory
allocator has some data structures too. What a fun to debug if this gets
damaged.

The POSIX standard defines a list of [async-signal-safe] functions. These are
the only (C) standard library functions you are allowed to use inside a signal
handler (there are some synchronous signal handlers, but let's not go into them
now). If you go through the list, you notice there's very little there ‒
specifically, no memory allocation and no locking.

### They are racy and hard to debug

Because signals are not part of the normal flow control, people tend to forget
about them. Depending on what you do, it is possible to write the code in a way
that loses some signals or waits forever for a new one even when it already came
‒ if you act on the signal and clear a flag in the wrong order and a new signal
comes just in between or something like that. Your first try will almost
certainly be subtly wrong, but work most of the time.

But you just can't write a unit test that hits the code in between two specific
instructions with a signal. It'll probably work differently if you put it into a
debugger or if you add logging around it, because of timing.

In other words, better write it correctly on the first try, because you won't be
able to fix it later on.

### They don't play well with threads

They were created at times when threads didn't exist. They have defined
interaction with threads today, but that mostly serves to know how bad the
situation is. A signal can run in mostly whatever thread it decides. It can run
concurrently. It (without reconfiguration) doesn't run recursively ‒ a second
`SIGUSR1` will pick a different thread to run on or wait for the first signal
handler of `SIGUSR1` to finish. But `SIGUSR2` still *can* interrupt that
`SIGUSR1` signal, or the other way around (even if it might be the same function
‒ if you were thinking about locking by a spin lock inside such signal handler,
better forget that too).

We probably want to touch some global state, likely mutate it (there's not many
ways to get any info out of a signal handler ‒ it doesn't have a return value).
We will access it from different threads, possibly concurrently, so we need to
synchronize it somehow. But remember, we don't have mutexes! Channels probably
have a locks and an allocation inside them too (the
[std](https://github.com/rust-lang/rust/blob/acf50b79beb5909c16861cc7c91e8226b7f78272/src/libstd/sync/mpsc/mod.rs#L830) channels,
[chan](https://github.com/BurntSushi/chan/blob/8093a9cfa7d614a320a0f6ada48ca6b6eb298d4b/src/notifier.rs#L42) and
[crossbeam-channel](https://github.com/crossbeam-rs/crossbeam-channel/blob/2317e6441afceb65c0f515d087603e8b24a8a637/src/internal/waker.rs#L209) do)

### The interaction with Rust is not defined.

The POSIX standard talks only about the C library primitives. So we either have
to use things from the [libc] crate only, or guess and audit the whole code of
whatever we use (and hope nobody adds an allocation in the next version ‒ adding
an allocation is certainly not considered a breaking change by most authors).
But, well, in the light of the above, we want to do as little of work inside the
handler as possible anyway.

While it is well defined to call a [longjmp] out of a signal handler, there's
no such mechanism in Rust. Panicking is somewhat similar, but that would
definitely be a panic across FFI boundary and that one is UB.

And, by the way, from the point of view of Rust, signals are definitely not a
safe thing ‒ if you can invoke UB by allocating or returning memory. I wonder
what other languages do about signals anyway, they probably have to account for
them deep down in their VMs.

## Why signals

By now I might have persuaded you that you don't want to touch a signal even by
a long pole when wearing a protective suit. But there are some times where you
probably have to.

If you write a daemon, you are expected to shut down correctly and gracefully on
several signals. You should reload your configuration on `SIGHUP` and reopen log
files.

If you start a subprocess ([fork]) and the subprocess ends, you'll get a
`SIGCHLD`. You are expected to pick up that corpse of the process or it becomes
a zombie. Unix is a rough world ‒ daemons, zombies, we recently acquired
spectres too.

## Common patterns

Signal handling is hard to do right, but there are very few things older than
signals still in active use. People developed patterns that were proven correct.
They usually work by smuggling the information a signal happened out of the
signal handler and then doing the needed work outside.

* Pretending they don't exist. This is very common, because most of the signals
  have quite sane defaults. Your program will terminate on `SIGTERM`. If you're
  not a daemon, but a tool to crunch some data, you just don't have to care.
  Rust already supports that out of the box.
* Ignore the restrictions and hope for the best. Signals don't come very often,
  so the chance of the deadlock or corrupt memory allocator internals is not
  that high… and there's a lot of software written in that way. But this is the
  wrong approach.
* Mask the signals in the whole program and use some mechanism to pull the
  pending signals out ([sigwait], [signalfd]). The first one needs a
  dedicated thread, the other one works well with some kind of IO event loop.
  But it is not applicable solution for a library, because the library has no
  way to mask signals in all the threads in an application (the application can
  mask signals before the threads are started and they inherit the mask).
* Perform only trivial actions. For example, exit the application.
* Make sure there's something happening often enough. Set a boolean flag in the
  signal handler and check it at each iteration. Rust doesn't seem to provide
  the C's `volatile sig_atomit_t`, but that one is for single-threaded programs
  anyway. It has [AtomicBool], which provides strong enough guarantees. Not
  all applications have this kind of periodic wakeup, though.
* Similar thing as above, but with clever masking of signals and unmasking them
  only as part of the IO waiting primitive ([psellect], [epoll_pwait]) ‒ the
  signal sets the flag and the primitive fails with `EINTR` error code.
  Not super-easy to write without race conditions, but it can be done. But
  we probably want to have several layers of abstraction between that and our
  application ([mio], [tokio]) and we don't have access to that.
* Implement a [self-pipe trick] ‒ have a pipe (or socket-pair or whatever) with
  both ends in the same program. Write one byte into it (non-blocking, in case
  it is full at the moment) in the signal handler. The read end can wake up
  whatever IO event loop the program has. You can have a pipe for each signal,
  or use the pipe just for wake up and add the boolean flags. A bit fragile for
  the correct order of actions around the read end and with some rough edges
  (spurious wakeups), but otherwise quite useful.

## Existing libraries

I didn't find much. There's [tokio-signal], but it is specific to tokio.
Others are either subtly broken (use allocation, mutexes), are very specific to
a single use case, don't allow sharing the signals, etc. In other words, I
didn't find anything I'd like to use except for [tokio-signal]. But I often
don't use tokio and having it just to handle the signals felt heavy-weight.

I didn't like writing the same boiler-platy unsafe code again and again, even
though I have some experience around signals. It's both tiresome and risky.
Therefore, I decided to write my own library.

## Goals for the library

I wanted to solve three problems, once and for all:

* Writing the same (or similar) unsafe code over and over again is tedious.
* Some of the above patterns are safe to use, so there should be a safe
  interface to it. You should not have to be a bearded Unix guru to shutdown
  your program correctly!
* Signal handlers are unique global resource ‒ there's only one signal handler
  of each kind. The library should be able to multiplex the handlers. If I'm
  successful here, other libraries that both use signals or expose some
  interface for signal handling would use it, so they don't step on each other
  toes.

## The implementation

There's one global [ArcSwap] with configuration at the very core of the
library. The configuration contains list of hooks to be called for each
different signal. The `load` method of `ArcSwap` is lock-free and doesn't use
any allocation, so it is safe to use from within a signal handler. Provided all
the hooks are also safe to call from within the signal handler and that we'll
not ever have to deallocate the configuration from within there, it is OK.

Replacing the configuration is done through an RCU trick ‒ it makes a full copy
of the old configuration, modifies it and installs it. But it makes sure to be
the last owner of the old configuration so all the deallocation happens outside.
This part is protected by a mutex ‒ it could be done without one, but it would
be significantly harder, as I want each signal handler registered on the first
use and only once. As registration and removal of hooks is more of a setup
thing, the performance doesn't really matter much.

The function to install arbitrary hook is obviously `unsafe`, because it could
put anything inside the signal handler. But the rest of the library contains
implementations of several of the above common patterns, with safe interface to
install that. So you can put an `Arc<AtomicBool>` flag inside that'll magically
turn to `true` whenever a signal is called, or a pipe that would be written
into.

There's also a higher-level API for signals. The structure implements
`Iterator` (actually, three different ones, depending on the exact needs). I'd
like to add support for [mio] and [futures] as well some time soon (I'll see
what is possible there without binding to specific runtime).

If you know of an useful pattern, you can either suggest it, send a PR to
the library or even implement a separate crate that just uses the multiplexer
(you save yourself the juggling with signal setup in addition to being able to
share the signal with others).

As an example, there's the other crate I wrote, [reopen], which helps with
reopening the log files on `SIGHUP`. Well, it provides more general interface,
so other things can be done too, but this is what I had in mind when writing it.
The library offers a convenient method to hook into a signal if compiled on
Unix-based system (and doesn't have that method on Windows).

## Stability

The API is not stable yet. While I don't plan any specific breaking changes
right now (and I don't see what they could possibly be on the core multiplexer
API), I want to get some bit of use and feedback before I commit to a specific
API.

## Thank you

I want to thank everyone for providing feedback and possibly help. But
specifically, I want to give thanks to two people. Martin Mareš, who introduced
me to the land of Unix and C and explained all the pitfalls there (that's one of
the bearded Unix Gurus I know) and Alex Crichton, who had done a very similar
thing when I sent a pull request to [tokio-signal] about a year ago.
Actually, a lot of inspiration and bits of the code of this library comes
directly from there.

[signal-hook]: https://crates.io/crates/signal-hook.
[async-signal-safe]: http://www.man7.org/linux/man-pages/man7/signal-safety.7.html
[real programmers]: http://www.jargon.net/jargonfile/t/TheStoryofMel.html
[tokio-signal]: https://crates.io/crates/tokio-signal
[libc]: https://crates.io/crates/libc
[sigwait]: http://man7.org/linux/man-pages/man3/sigwait.3.html
[signalfd]: http://man7.org/linux/man-pages/man2/signalfd.2.html
[psellect]: https://linux.die.net/man/2/pselect
[epoll_pwait]: http://man7.org/linux/man-pages/man2/epoll_wait.2.html
[self-pipe trick]: https://cr.yp.to/docs/selfpipe.html
[mio]: https://crates.io/crates/mio
[futures]: https://crates.io/crates/futures
[AtomicBool]: https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html
[tokio]: https://crates.io/crates/tokio
[longjmp]: http://man7.org/linux/man-pages/man3/longjmp.3.html
[fork]: http://man7.org/linux/man-pages/man2/fork.2.html
[reopen]: https://crates.io/crates/reopen
[ArcSwap]: https://crates.io/crates/arc-swap
