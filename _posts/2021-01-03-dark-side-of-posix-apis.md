# Dark side of POSIX APIs

One user of the [signal-hook] crate expressed an interest in knowing the PID and
UID of the process that sent the signal. As a general idea I liked it. The
low-level [`sigaction`] does indeed provide this information and it might be
useful (more than once I have wondered why the program terminated because
*someone* sent it a `SIGTERM`, but the logs contained no information about what
other process it was; knowing its PID is the first step to figuring it out,
though I don't know of a nice portable way to convert a PID to a name of a
process or something of similar usability). In fact, I had a long-standing open
issue on the repository about being able to somehow make the additional
information in the `siginfo_t` structure available.

So I've decided to give it a go and implement it. In fact, I've already had the
place in the API this would plug in that would make it immediately available to
all the `signal-hook-*` asynchronous add-ons too, not only the iterator inside
`signal-hook`.

For a few reasons, this turned out to be a bigger challenge than I expected.

## The usual async-signal-safe trouble

Inside the signal handler, one can do almost nothing. It's because it can happen
on whatever thread at any time, so one can't allocate in there (what if that
thread was actually in the middle of some allocator routine with inconsistent
thread-local data structures), can't lock stuff (the lock could be held by the
same thread, which would deadlock)... actually, there's a
[list of functions][async-signal-safe] that can be used from the C standard
library, but calling anything else is UB.

So [signal-hook] gives the user bunch of patterns how to postpone the reaction
to a signal for a later time, for once outside of the actual handler. There the
user can enjoy the full feature set of Rust or actually anything one can call
from it too. But it's one thing to export „This signal happened at least once“
and another „This signal with _all this auxiliary information_ happened“.  For
one, collating multiple instances of the first is losing much less information
than in the latter case. For another, the first is a single bit of information,
here we have much more.

For obvious reasons, having something like a global `Mutex<Vec<Info>>` doesn't
work (locking, allocations, ...). Having a static mutable variable doesn't
either, because the same signal can happen on two different threads at the same
time (thank you, POSIX!, you're not making it any easier).

The good news is, atomic variables work. The less good news is, all the
information I wanted to take out was more than 64 bits. And some of the
platforms signal-hook has in its CI don't even have 64bit atomics! So encoding
everything into a 64bit number is not the right way.

I've had to implement a kind of [channel-like thing][channel]. It has some
preallocated slots to store the information in and few atomic variables around
that to make sure two threads never write to the same one and that the reader
knows which slots to read things from. They do kind of locking, but instead of
blocking, the channel throws items out of the window if it is full. There's
nothing much better it could do, clearly it is not possible to allocate and if
the reader end doesn't keep up, any kind of preallocated capacity will get
exhausted eventually.

Also, the signal only provides storage of the data, not any kind of wakeup (but
other parts of that crate already supply that).

## Reading data out of `signinfo_t`

So the kernel gives us a nice `siginfo_t` structure we can read any and all the
extra information we are interested in. But there's a catch. The POSIX standard
gives *a lot* of room for interpretation of how that structure should look like.

Citing from the manual page of [`sigaction`]:

> si_signo, si_errno and si_code are defined for all signals.  (si_errno is
> generally unused on Linux.)  The rest of the struct may be a union, so that
> one should read only the fields that are meaningful for the given signal:

OK, so if I'm reading this correctly, we get this:

* `si_signo`, which is the same as the signal number we already got. So not
  really useful.
* `si_errno`, which is useless on Linux at least (might be useful somewhere
  else).
* `si_code` which tells us what caused the signal to happen (was it sent by the
  kernel, or some process, or was it caused by a timer, ...). We actually *need*
  that to access some other fields, otherwise they might not be there because of
  some union style optimization to make the *temporary on-stack* structure a
  tiny bit smaller. If we access them and they are not there, it may even be an
  UB (who knows at this point, it's interaction of C and Rust, better not even
  try).

We *would* be interested in the `si_pid` and `si_uid` fields. According to the
[`libc::siginfo_t`] documentation, there are methods for these (not fields,
because of `union`s). And they are `unsafe`, apparently because we first have to
check the `si_code` (their unsafety is not explicitly documented, but one can
assume).

### The `SI_*` constants

Long story short, while Rust's `libc` bindings give one access to the `si_code`
field, it doesn't export the actual constants to know what that value means.
While the maintainers are generally open to adding these constants, I don't have
access to all the large number of platforms to figure out what values the
constants have on each (yes, they are not the same) and what constants are even
available on each of them (for example the `SI_KERNEL` seems to be available
only on Linux).

If you get the idea in the lines of „Well, I'm interested in `SIGINT`, where
else could that come from than from other process“, then it's worth mentioning
that if you press `CTRL+C` in a terminal, it is the kernel that sends the signal
on behalf of that PTY, not the terminal emulator. At least on Linux, you'll get
`SI_KERNEL` and no PID or UID, therefore that idea is maybe leading to UB.

Furthermore, the `si_pid` and such methods are not available on all the
platforms [signal-hook] compiles on.

So while the theoretically *right* way to move forward would be to go over a lot
of platforms and extend `libc` with each of them, it would take far more time
than what I have available. As a workaround, I've decided to write a minimal
extraction code in C (which has access to the actual platform C headers,
including the ability to detect if some constant exists by `#ifdef`s).

As having a working C toolchain available may be a bit of a pain (certainly in
some cross-compiling situations), and it adds another dependency ([`cc`]), this
decoding ability is gated behind a feature flag.

### MacOS and the `si_code`

To add insult to injury, apparently on MacOS the `si_code` field is there, the
constants are there, but the kernel *does not set them*. The field is always 0.
As a small compensation, the structure is not an union, so we *can* read the
fields of that structure no matter how the signal happened. In case there's no
information about a process, both these fields are set to 0.

So in an effort to unify these things a little bit, there's a special case for
MacOS to deal with this. Users of [signal-hook] don't need to worry about this…
well, it probably is technically not a violation of the POSIX standard, the
standard is just implemented in a very not useful way. But parts of the
information won't be available on MacOS.

### How reliable it is

I've done manual testing on Linux, the CI does some automated testing on few
platforms (actually checking the value of the extracted PID) and compiles on
many more. So it should work.

But because some information is platform specific, generally unknown situations
are reported as `Unknown` and it is possible the kernel or the library
consolidated multiple same signals into one, the extracted information should be
mostly informative (interesting info in logs, for example). That is, if one gets
some information, it should be correct, but it is not guaranteed to always get
useful stuff.

## The results

Now you can get the information about where the signal came from, something like
this:

```rust
use signal_hook::consts::TERM_SIGNALS;
use signal_hook::iterator::SignalsInfo;
use signal_hook::iterator::exfiltrator::WithOrigin;

type Error = Box<dyn std::error::Error + Send + Sync>;

fn main() -> Result<(), Error> {
    let mut signals = SignalsInfo::<WithOrigin>::new(TERM_SIGNALS)?;
    for signal in &mut signals {
        dbg!(signal);
    }
    Ok(())
}
```

With first CTRL+C in terminal, then sending a signal with `killall`, the output
would look something like this (yes, it would be nicer if that signal got
translated to something like `SIGTERM` instead of the number, I'll see about
that in the future):

```
[src/main.rs:10] signal = Origin {
    signal: 2,
    process: None,
    cause: Kernel,
}
[src/main.rs:10] signal = Origin {
    signal: 15,
    process: Some(
        Process {
            pid: 25540,
            uid: 1000,
        },
    ),
    cause: Sent(
        User,
    ),
}
```

## Conclusion

When dealing with C++ APIs, one expects that C++ can express some things that
are not reasonably easy to translate to Rust and may need some kind of C++-side
wrapper to make it usable.

It seems C and the POSIX standard has some of these dark corners where the best
option is to adapt a similar strategy too. It's not too hard to pull it off, but
it's a bit extra work and relies on having a C compiler available.

[signal-hook]: https://crates.io/crates/signal-hook
[`sigaction`]: https://www.man7.org/linux/man-pages/man2/sigaction.2.html
[async-signal-safe]: https://man7.org/linux/man-pages/man7/signal-safety.7.html
[channel]: https://docs.rs/signal-hook/0.3.2/signal_hook/low_level/channel/struct.Channel.html
[`libc::siginfo_t`]: https://docs.rs/libc/0.2.81/libc/struct.siginfo_t.html
[`cc`]: https://crates.io/crates/cc
