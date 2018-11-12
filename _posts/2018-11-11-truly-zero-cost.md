# Truly zero cost

I know it is claimed how Rust has zero cost abstractions and such and that all
these levels of abstractions will just go away in a release build. But there's a
difference in hearing the theory and seeing it really happen in practice. And I
don't appreciate it because I'd consider it magic, but more because I understand
how that is being done and it *still* looks cool.

## The background

I wrote this [arc-swap] library that allows storing an [Arc] and swap it
atomically while other threads keep reading it. It's something like
`RwLock<Arc>`, but faster and readers are never blocked. And, from time to time
I try to polish it a bit more and make it a bit faster under this or that
circumstance. Not that there would be some great need for that, but some people
like fast cars, other people like playing with hard problems, it's probably
similar.

Anyway, there's this thing I call a generation lock that is being used
sometimes. It's a one-sided spin-lockish thing. While readers never get blocked
by anything, writers *can* get blocked by this. The [arc-swap] library offers
several mechanisms how to read the data, some of it sometimes using this
generation lock. And usually even when using the generation lock, it's locked
for a really short amount of time, so it's not an issue. However, I also use it
in the [signal-hook] library, where there's not much choice ‒ allowed operations
inside a signal handler are severely limited and the generation lock is the only
option available. And the signal handler might hold it for longer.

That in itself would be mostly OK, except that performance is always about
trade-offs. So, because the [arc-swap] tries to be fast on read operations even
if many happen at the same time on the same `ArcSwap`, I had multiple copies
(shards) of the generation lock, spreading the load and lowering the contention
over the cache lines where the lock lives. But because that takes a lot of
space, I had only one *set* of the locks shared by all the `ArcSwap` storages.
This means that the signal handler could block other, potentially
performance-sensitive parts of application from doing their thing and for a long
time ‒ depending on how much the signal handler was doing. Not good.

## The goal

Obviously, there's the same tool that can be used for two very different
purposes. One that needs to be fast, the other that simply needs to work in
restricted environment (the signal handlers), but without much performance
needs. They are slightly in conflict with each other on some little details. The
signal handler thing would really like its *own* generation lock. It doesn't
care about the contention and it doesn't care if that *single* `ArcSwap` will be
few bytes larger. But most of the code is the same.

I could have taken the code, stripped it of all the fancy things I didn't need
for the signals and embedded this stripped down thing into [signal-hook]. But
that didn't feel right ‒ all that code duplication and stuff, most of the code
would be identical.

## How it went

Rust traits are like the hammer that makes everything look like a nail.
Therefore I went and defined a trait to describe the place where the [lock is
stored][LockStore]. Then I made the `ArcSwapAny` (a „backend“ type with many
parameters which is used by type aliases like `ArcSwap`) generic over the
`LockStore`. Adding a type parameter can actually be a semver-compatible change,
if a default is provided.

Currently, it can either be a zero-sized field that delegates to the global
storage as before, or a field that actually keeps the generation lock inside
itself.

The places in the hot-path code where I was directly accessing a global
fixed-sized array got replaced by calls to methods on some generic storage,
returning some associated type that could be turned into a reference to a slice.

From what the code was saying to do, it got changed from:

* Pick `n`th element of the global array on address `0xABCD`. That's probably
  something like one or two assembler instructions.

To:

* Look into `self` and take a reference to this field inside.
* Call a method on this field.
* Take a reference to that global array on address `0xABCD` and return it.
* Call `as_ref` on that reference, turning it into another reference to a slice
  (actually a fat pointer, so adding a length).
* Pick `n`th element of it, checking against the length stored in the slice.

That looks like a *lot* of extra work to do.

But I'm pretty sure it got just thrown away by the compiler and turned back to
that one or two assembler instructions. Not that I'd be reading the assembler
code that got out ‒ I'm old school, but not really *that* old school to read
pages of machine code before bedtime. But as I said, the library cares a lot
about performance, so every time I touch part of the hot code, I re-run the
benchmarks. This way, I throw out about half of the optimizations again (as they
make it actually slower). And the benchmarks are *very* sensitive to even tiny
changes ‒ adding length check for the slice would *probably* turn into visible
change in the numbers.

I'm also pretty sure that most languages would not go that far. The idea that
the type plugged in has only one possible value, therefore it doesn't need to be
stored and methods on that don't care about the `self` reference is pretty neat.

On the other hand, it's kind of mind-bending, considering how much work there
needs to go into *throwing away* most of the code I've written. There's an arm
race going on between the compilers, competing in how much code they are able to
discard.

## Hints for the compiler

Actually, I *did* try make it as easy for the compiler as possible. The trait
implementation has `#[inline]` stuck on the methods, so the optimisations can
happen across the function boundary. This doesn't always help and sometimes even
hurts, but this time it turned out for the better (well, at least not hurting).

The other thing is, I'm actually declaring the true type the methods are
returning as an associated type, not returning a slice-reference. During the
measurement, I was seeing this to have a small effect. But I can't say if it was
large enough to be significant, or if it was just a noise. Anyway, preserving
critical information ‒ like a constant length of something ‒ across abstractions
helps the compiler in doing its job.

[Arc]: https://doc.rust-lang.org/std/sync/struct.Arc.html
[arc-swap]: https://crates.io/crates/arc-swap
[signal-hook]: https://crates.io/crates/signal-hook
[LockStore]: https://github.com/vorner/arc-swap/blob/master/src/gen_lock.rs#L68
