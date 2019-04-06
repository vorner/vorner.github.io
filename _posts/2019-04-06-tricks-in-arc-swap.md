# More tricks up in the ArcSwap's sleeve

This is a continuation of the [Making Arc more atomic][original] post. In short,
[ArcSwap] is a place where you can atomically store and load an [`Arc`], similar
to `RwLock<Arc<T>>` but without the locking. It's a good tool if you have some
data that is very frequently read but infrequently modified, like configuration
or an in-memory database that answers millions of queries per second, but is
replaced only every 5 minutes. The canonical example for this is routing tables
‒ you want to read them with every passing packet, but you change them only when
routing changes.

I've been asked a few times how it works inside and I don't want to repeat
myself too much, so here we go. This is the high level overview of what happens
under the hood, you'll have to read the [code][repository] for the fine details,
proofs of correctness, etc.

I'll be talking mostly about [`Arc`], but the thing is generic and can be made
to work with other things ‒ notably, `Option<Arc<T>>` is supported by the
library, and if you have your own version of [`Arc`], you can implement a trait
on it and there you go too.

## The devil is in the details

In the heart of the thing there lies an [`AtomicPtr`]. As [`Arc`] can be turned
into a raw pointer and created from one, it can be stored in there. That
conversion keeps the reference counts. The [`swap`] method is therefore easy ‒
one pointer with its reference goes in, another gets out. Just make sure the
atomic orderings are correct. [`store`] is just a swap that throws the result
out. That's easy.

What does some kind of [`load`] look like? It needs to read the pointer, and
because it returns one [`Arc`], while keeping another „virtual“ [`Arc`] inside
the storage, it needs to increment the reference count. But here lies the
problem, as by the time we get to incrementing it, some other thread might have
swapped the pointer out of the storage, dropped its copy, reached zero refcount
and deleted the data, including the reference count itself. Ouch. Not good. We
can't increment non-existing reference count.

So, the original solution was a kind of uni-directional short-term lock which
prevented the writer (the one doing [`swap`]) from completing before we
increment the refcount. Read the [original] article about the details, it is all
about that lock.

## Properties of the lock

The original one (now called *generation lock*) had some good properties.
Specifically, the read operation was [wait-free], which is great. Literally, a
reader stopped for nobody, performed its fixed sequence of operations and
continued with a copy of the [`Arc`] in its hand.

There were just three downsides:
* While the lock is able to deal with constant stream of readers and a writer
  will finish even in face of that, if a reader gets *stuck* in the critical
  section, all writers wait forever.
* Even though the read operation is [wait-free], all readers write into the same
  cache line ‒ specifically, the lock *and* the reference count. So if there are
  many concurrent reads, they get slowed down because CPUs fight over which one
  of them would be writing into the cache line, making it *quite* slow (well,
  here we are talking on scales where going to fetch data from RAM is considered
  *eternity*).
* The storage for that pointer was bigger than the size of a pointer, inflating
  any data structures built of these.

## Sharding the lock

For this library, read performance is much more important than write
performance. Therefore, the first changes I've made were to use multiple
instances of the lock. Each reader would pick just one of them and lock it,
while a writer would have to wait for a permission from all of them. By making
sure the locks align to a multiple of cache lines size, the chance of two reader
threads colliding on the same cache line is lowered (while making it slower for
writers).

However, this doesn't solve the contention on the reference count. If the only
thing I want to do is to read one integer from configuration behind that
pointer, incrementing and immediately afterwards decrementing the shared
reference count is a big deal.

So I've made another method, the [`peek`], available. Instead of locking,
incrementing the reference count and unlocking, it returns the lock guard with
access to the data. The caller reads that `i64` from there and unlocks, never
getting the actual [`Arc`] created (but keeping the data alive for the whole
time) ‒ and never touching the reference count.

This solves the problem for very short reads, but runs the risk of making the
writer wait in a busy loop for longer if my [`peek`] into the data is not so
short. I could still take the [`load`], which creates the [`Arc`] with its
reference, but that feels wrong.

## Making the lock configurable

By default the storage uses a *global* set of locks. The idea is, if a thread is
busy locking one `ArcSwap`, it won't be trying to lock a different one at the
same time. That way, `ArcSwap` itself has the same size as a pointer, but runs
the risk of blocking writers on *unrelated* instances of `ArcSwap`.

It is possible to fine-tune the behaviour by a type parameter. The library
contains implementation of private locks for each separate instance. This is
even used inside the [signal-hook] crate. There, this lock is the *only* allowed
possibility due to restrictions what can be done inside a signal handler.
However, the signal handler may run for a longish time and it would not do to
block unrelated processing. With a private lock instance, only adding or
removing signal handlers is affected (which usually happens just on application
startup).

## What about hazard pointers

As you can see, the lock approach isn't perfect.

I'm certainly not the only person to take an interest in a similar problem. I've
came across something called [hazard pointers] and one specific implementation,
the [`AtomicArc`] was certainly a very interesting read. Unfortunately, as far
as I know, it's not yet finished and released.

On a very *very* high level, we have a global „don't you dare to free this, I'm
using it right now“ registry. If I want to start doing something with a pointer,
I read it, write it down into the registry and check by reading it once more (to
avoid races). Only after this I can actually access the memory. When done, I
remove it from the registry.

If I want to free a pointer, I first remove it from the storage (so nobody else
can read it any more), then walk the *whole* registry. If I don't see the
pointer mentioned there, fine, I can safely free it. If it *is* there, I mark it
in the registry with a note „you're now responsible for freeing it“, passing the
problem onto some other unfortunate thread. That thread in turn will try to free
it (with the same check through the whole registry) when removing its entry from
the registry.

This is all very nice and shiny, but I didn't like two details about it:

* Searching for a free spot in the registry can take a long time. Additionally,
  a reader thread may become responsible for freeing the pointer, forcing it to
  walk the registry. I want my readers to be fast (and wait-free, not *just*
  lock-free). Like, always, not just most of the times. Not for some actual
  reason, it is simply a matter of personal pride.
* While the *idea* is relatively simple, code that handles all the corner cases
  in the registry in face of concurrent lock-free access (it wouldn't make much
  sense if we put locks into the registry, would it) turns out quite tangled.
  Combined with the general inability to write unit tests for these corner cases
  (they happen as a result of thread scheduling, which CPU is faster, etc), this
  feels like a dangerous path to take. I do enjoy hard problems and such, but
  there are certain limits, especially if I want to claim some confidence about
  correctness of the thing. Furthermore, all the „cleverness“ of the code slows
  it down visibly in benchmarks.

## The hybrid approach

This is the implementation of the third loading method, [`lease`]. This one
should probably be considered the primary one for most use cases.

Eventually I've decided to trim the hazard pointers down to a bare minimum. If
anything *at all* doesn't go according to the best-case scenario, it falls back
to a full load instead. That one is still formally wait-free and by making it
*rare* to happen, the problem with contention on the reference count should be
effectively gone.

Furthermore, the responsibility for freeing something is not passed through the
registry. A record in the registry means there's an [`Arc`] somewhere which owes
one reference (that is fine *as long as we don't reach 0*). For this reason, it
is no longer called hazard pointers in the code, they are *debts*. So instead of
passing the responsibility to walk the registry, the writer takes the debts onto
itself (removes it from the registry) and pays them (by incrementing the
number). Actually, it pre-pays one upfront and removes it at the end, to be on
the safe side.

The read operation therefore tries this, in order:
* If this is the first access from the current thread, set up fixed number of
  slots in the registry. They are going to be for the exclusive use by my
  thread (well, the code is ready for potential sharing of the slots between
  threads, but no kind of slot stealing is implemented and probably not worth
  it). Only this part is just lock-free because the new slots need to be linked
  into a linked list and there may be retries doing that. Future reads from
  this thread will be wait-free.
* Read the pointer.
* Search for an empty slot through my fixed number of slots. If not found, fall
  back to full load. If found, register the debt.
* Re-read the pointer. If it's the same, we're done.
* If it is different, try to remove the debt from the registry. If it was
  already paid by someone else, we have a full [`Arc`] in our hands, which
  is great and we are done. If not, fall back to full load.

Just not to confuse people reading through the code, the [`Lease`] guard does
not store the full [`Arc`] as an [`Arc`], only as the raw pointer and that
[`Arc`] is only virtual (eg. there's the reference count for it).

When dropping the guard, we check if we owe a reference. If we do, we actually
have *nothing to do* (except for removing it from the registry) ‒ we simply make
it even by stopping to exist. If we don't (either because we've done a full load
or because someone paid our debt), we decrement the reference count.

As long as the current thread doesn't hold more leases than there are slots,
this is pretty fast ‒ comparable to lock & unlock of *uncontended* [`Mutex`]
(Muteces are slow only if locked, because then they have to wait in line, and go
to the OS to put the thread to sleep…), but with the ability for arbitrary
number concurrent readers ‒ no *risk* of being locked or slowed down.

## Benchmarks

The repository contains some benchmarks. The ones I consider the most relevant
are doing the measured operation (eg. [`peek`]) while some other threads do
„background noise“ traffic ‒ like writing new values all the time. There are
many combinations of the measured operation and how many threads do the
background traffic and what kind of background traffic. This is done with
several alternatives to compare with [`ArcSwap`].

However, benchmarks are not the real thing, for many reasons. First and
foremost, real applications do other stuff than just access one shared piece of
data, therefore the differences might be hidden in the costs of whatever else
the application does. Apart from that, it turns out CPUs are sensitive to mostly
*whatever*. Like, spinning up a parallel thread that *doesn't* access anything
near our data nevertheless may slow our core down because of the temperature
gets higher or because the power subsystem is not able to feed both. So while
interesting, benchmarks are not to be taken *too* seriously.

## Areas of improvement

I'm open to ideas what and how to improve. In particular, I believe there is
room for improvement in the writer performance (that one is rather slow) ‒ but I
want to do that only if reader performance doesn't get hurt. This crate is
biased towards readers, if you need something „balanced“, there's a place for
some other implementation.

Also, discovering places where the documentation is not perfect (and sending a
PR) would be highly appreciated. Or coming up with a nicer API.

If you feel like reading the proofs and the code and double-checking my
reasoning is sound and that I didn't overlook something would also be very
valuable.

## Implementation for other languages

I've already been asked if I would like to port this to another language
(specifically, C++). My answer to that is *no, I wouldn't*. This code is
*already* quite challenging even with all the help and checks I get from the
Rust compiler (which is a lot even if the code is using `unsafe` extensively).
And I don't have the free time for that. Besides, I don't think C++ `shared_ptr`
can be turned into a *single* pointer.

However, if *you* want to go ahead and implement it in some other language, I'll
be more than happy to answer any questions and walk you through the design or
just talk about it.

[original]: /2018/06/24/arc-more-atomic.html
[ArcSwap]: https://docs.rs/arc-swap
[`Arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html
[repository]: https://github.com/vorner/arc-swap
[`AtomicPtr`]: https://doc.rust-lang.org/std/sync/atomic/struct.AtomicPtr.html
[`swap`]: https://docs.rs/arc-swap/0.3.*/arc_swap/struct.ArcSwapAny.html#method.swap
[`store`]: https://docs.rs/arc-swap/0.3.*/arc_swap/struct.ArcSwapAny.html#method.store
[wait-free]: https://en.wikipedia.org/wiki/Non-blocking_algorithm#Wait-freedom
[`peek`]: https://docs.rs/arc-swap/0.3.*/arc_swap/struct.ArcSwapAny.html#method.peek
[`load`]: https://docs.rs/arc-swap/0.3.*/arc_swap/struct.ArcSwapAny.html#method.load
[`lease`]: https://docs.rs/arc-swap/0.3.*/arc_swap/struct.ArcSwapAny.html#method.lease
[`AtomicArc`]: https://github.com/stjepang/atomic/blob/master/src/atomic_arc.rs#L20
[hazard pointers]: https://en.wikipedia.org/wiki/Hazard_pointer
[`Lease`]: https://docs.rs/arc-swap/0.3.*/arc_swap/struct.Lease.html
[signal-hook]: https://docs.rs/signal-hook/
[`Mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
