# Making `Arc` more atomic

This is a story of a tiny feature I was missing in Rust… so I created it (partly
because I like the feature, because it felt wrong for Rust not to have it, but
mostly for the practice and fun of beating a hard and interesting problem). You
can read the story if you are interested about the behind the scenes, about the
feature itself, how to use it or just for fun ☺.

## The missing feature

There are two smart pointers that allow having a shared ownership in Rust,
`std::rc::Rc` and `std::sync::Arc`. `Rc` means „reference counted“ ‒ it keeps a
pointer where both the value and a reference count lives. When the pointer is
cloned, the reference count is increased, when one pointer is dropped, the
reference is decreased. If it reaches 0 (nobody points to it any more), the
value is destroyed. There are some more details (cycles, weak counts ‒ you can
read the [documentation](https://doc.rust-lang.org/std/rc/struct.Rc.html)).

The `A` in `Arc` stands for „atomic“. It allows all these increments and
decrements to work correctly across threads (`Rc` plain out refuses to have
anything to do with more than one thread) ‒ so more than one thread at a time
can point to the same value and no matter which thread drops the last pointer,
the value gets destroyed then and only then correctly.

From this description, it looks a lot like `std::shared_ptr` from `C++`.
However, the `shared_ptr` has one extra capability the Rust's `Arc` doesn't. In
Rust, only the reference count is atomic (and safe to manipulate from multiple
different threads), but the `Arc` itself isn't. That means that if we want to
change *where* one given `Arc` points (not just clone it or have it in multiple
threads), it needs to live under a `Mutex` or something similar. In `C++` there
are `std::atomic_load` and `std::atomic_store` functions that have
specializations for `shared_ptr` and allow treating the pointer itself
atomically.

I decided it would not do for Rust to lack such a great and needed
functionality.

## What good is it?

Sure, if `C++` has the ability, so Rust should not stay behind, but what is the
actual use case? I myself know of two.

First, a niche one, is that `Mutex`es are not allowed inside Unix signal
handlers and one needs to synchronize data, because signal handlers can be run
in whatever thread the OS likes (well, yes, there are ways to have a say in
that, but still). Therefore, having *something* that can be used atomically
instead of behind a mutex from within a signal handler is of some help ‒ it
doesn't make handling signals in non-trivial ways easy, but at least possible
(more on that one in a future writeup).

The second one is form of RCU-like (Read-Copy-Update) data structure. It can be
implemented by other means, but one way is with an `Arc` that changes from time
to time.

Imagine a server that answers queries about some data set (maybe a DNS server,
or maybe it's not even a data set, it's a configuration of your program). The
data set is in the RAM, so the answering is fast. But from time to time, the
data needs to be updated ‒ let's say read from a file. We could lock the
data structure and update it ‒ but we could not answer queries while we are
updating it, creating a short downtime for the server. Depending on the kind of
server, it might or might not be a problem.

Or, we could have the data set inside an `Mutex<Arc<D>>`. The query would lock
the mutex, clone the `Arc` and unlock. Similarly, the update would prepare the
new data set, lock the mutex and exchange the pointer and unlock it. This makes
the downtime really short, the heavy updating is done outside of the lock. We
could actually use `RwLock` instead of `Mutex`, which allows multiple queries to
make the copy of the `Arc` at the same time. That's pretty good (probably good
enough for 99% practical uses), but it still has some downsides. It places
locking onto the fast path of queries, whenever there's an update, the queries
have to wait for a short time and a steady stream of queries can block updates
forever in some implementations of `RwLock`. The code could look something like
this:

```rust
lazy_static! {
    static ref DATA: RwLock<Arc<Data>> = RwLock::new(Arc::new(Data::load()));
}

fn main() {
    for _ in 0..WORKER_THREAD_CNT {
        thread::spawn(|| {
            loop {
                let query = read_query();
                let data = Arc::clone(&DATA.read().unwrap());
                let reply = data.answer(query);
                send_reply(reply);
            }
        });
    }
    thread::spawn(|| {
        loop {
            thread::sleep(Duration::from_secs(900));
            let data = Arc::new(Data::load());
            *DATA.write().unwrap() = data;
        }
    });
}
```

If we had an `Arc` that could be loaded atomically (without locking or waiting)
and changed atomically, we could do even a bit better, avoiding all the locks on
the hot path and nothing could ever get blocked.

## Ingredients

We have an `Arc` as mentioned above. That one can track how many pointers point
to our data. But it can't be loaded or stored atomically.

There's also `AtomicPtr`, which allows loading and storing atomically. But it
does not track how many pointers point to the data, therefore we don't know when
to free the data. We are not in a garbage collected language, this is a problem.

I've decided to crossbreed them. The result is the
[`arc-swap`](https://crates.io/crates/arc-swap) crate and the `ArcSwap` type,
which can store an `Arc` into itself and provide a copy, both in a lock-less
fashion.

## Challenges

I wanted the atomic storage to work with ordinary `Arc`s from the standard
library, not to build my own, incompatible, smart pointer ‒ too many things in
Rust operate on the standard one. Therefore, I had to work with whatever the
public interface gives me.

It has the `into_raw` and `from_raw` pair of methods. When storing the Arc,
we can turn it into a raw pointer and that in turn can be stored inside a
`AtomicPtr`. So far so good.

The `into_raw` method does not decrease the reference count ‒ even in the raw
form, the pointer still holds its shared ownership ‒ and so does the
`AtomicPtr`. Basically, the `AtomicPtr` is a frozen storage for the `Arc`.
We just need to build our operations so they preserve the correct reference
counts. Creation of the `ArcSwap` is easy, because there's no chance of
concurrency yet.

Let's do `swap` ‒ exchange the content with a provided `Arc` and return the
original. The `swap` preserves the reference counts, so all is well. And it can
serve as the building block for `store` ‒ it swaps and throws the original away
(in the `Arc` form, which will take care of all reference count decreasing for
us).

The problem lies within `load`. Because we would like `load` to return a new
`Arc` with the pointer, but also keep the original stored, we need to increment
the reference count somehow ‒ `from_raw` claims ownership of the one reference
inside the `AtomicPtr`, we need to generate a new one for it. We can do it by
cloning and turning one clone of the `Arc` back into the raw pointer (and
throwing it away).

However, while `load` from the `AtomicPtr` is atomic, we now have a short time
before we read the pointer and increase the reference count (there are some
papers how this could be done atomically with double-word atomics, but it would
not work with the standard library `Arc` even if we had the double-word
atomics). If some other thread managed to `swap` the original pointer out and
drop it before we increase the reference count, we could get to 0 prematurely,
and get our value freed. That's not good.

## Intermezzo with `C++` standard library

I went to read the `C++` standard library code (the one coming with gcc),
because, after all, they already implemented it. I wanted inspiration.

Leaving aside that Rust's standard library is much more readable than the `C++`
one (imagine a forest of templates and everything named with two leading
underscores), I did not find my inspiration there. Instead, I found several
things with `mutex` in their name around `shared_ptr` and its
`atomic_load`/`atomic_store` function specializations. As it turns out, `C++`
standard does not mandate that these functions are lock-free.

## Protecting the dangerous area

The next step was patching the dangerous area where we owe the reference. The
initial fix was to add another atomic variable (`AtomicUsize`) inside the
`ArcSwap` that would count how many threads there are inside `load`. The `swap`
could then wait until the count drops to 0 before returning the old pointer to
its caller. That way it would make sure no `load` owes the reference to the old
value (another future `load` could own, but to the new value). And bad things
can only happen when we drop the `Arc`, which happens outside of `swap`.

We prevented a crash, but it's still not good enough. The problem is, if the
`ArcSwap` is hammered enough with `load`s, it would never drop to 0 and the
`swap` would wait forever more (and busy-wait on reading the counter).

The trick is, the `swap` doesn't have to wait for no `load`s to be present in
the dangerous section. No *old* `load`s that have seen the previous value is
enough. Tracking the `load`s by which exact version of the pointer they've
seen would be expensive. But we can make an error in the safe direction, by
allowing to recognize it a bit later.

Therefore, we have two counters for the tracking of the `load`s and a switch to
route new ones to either one or the other counter. If they are routed to the
first, the second will eventually drop to zero. A waiting `swap` is allowed to
turn the switch if it sees 0 in one, so the other one might get empty later on.
Having glimpsed a zero in both (not necessarily at the same time) means any old
load that saw the previous value must have gone away ‒ it must have been in
either one or the other counter.

This way, the `swap` still has to wait for the `load`s to clear, but there's
guarantee at least one `swap` makes a progress each time there's a zero
(possibly more, but that much is not guaranteed). And multiple `swap`s can wait
at the same time (same as multiple `load`s can happen at the same time). Seeing
the generation increase by at least 2 should be enough too in theory (some other
thread seeing the 0 also means it happened) but I have yet to implement it
in a way this is faster in practice.

A load is always the same sequence of instructions. It will still slow down a
bit if multiple loads happen at the same time, though, because the CPUs need to
fight over the same memory location (there's no lock-contention on the program
level, but there's contention on the HW level).

In the jargon, `load` is lock-free and wait-free. ~~The `swap` (and therefore
`store`) is only lock-free, but that still allows it to be used inside a signal
handler (however, signal handlers have memory allocation and deallocation
forbidden too, so making an actual use of that is hard) and in
massively-parallel workloads.~~

Edit: It turns out `swap` is *not* lock-free and can't be safely put inside a
signal handler (without some special care) ‒ `swap`s won't block each other, but
a interrupted `load` can block it. I'll see if something can be done about that,
but at least `load` can still go into the signal handler (which is what my
signal use case is about).

## Doing the true RCU

The example above assumed the whole data set was loaded completely anew each
time and simply replaced the old value. Sometimes, we want a true update ‒ take
the old data, make modifications to it and store the update. There are two
reasonable ways to do that.

The simpler one is probably making sure there's only one thread doing updates.
That way, this thread can read the old value, create the updated version and
swap it without fear it changed in meantime. This can be inside a mutex or by
having a dedicated update thread.

If there could be multiple update threads, one would generally do the same, but
retry the update (or relevant part) if the pointer was changed by another thread
in the meantime.  The `ArcSwap` has the needed `compare_and_swap` method to
implement this and an `rcu` helper method doing these retries.

There's also a little devil in the details. The last thread holding the original
value is responsible for dropping the whole data set. If the data is expensive
to drop, it could slow down the query and we are doing this thing to have fast
queries. The updater thread could make sure it is the last one holding it (it
can afford the slowdown), by repeatedly doing `try_unwrap` on the `Arc`, until
it succeeds (there's a helper method on `ArcSwap` for this thing too). The
example code above does not suffer this problem, because the answer is sent
before the drop.

The update might be done in several ways:

* Cloning the old value and modifying it. This is simple, but slow and consumes
  twice as much memory (in case of large data sets).
* If operating in the single thread, it could hold one „production“ copy and one
  „standby“ copy. It makes the update to the standby and swaps them. Then it
  makes sure it is the only owner (by the `try_unwrap` trick from above) and
  makes the same update to it too. It is now ready to accept a new update. This
  still makes use of twice as much memory, but the updates can be much faster,
  because there's no need for the cloning.
* Using persistent data structures. These are usually some kind of trees built
  from `Arc`s. An update needs to clone only the one traversed path of the tree,
  making both the update and drop of the old version much cheaper ‒ they share a
  lot of unchanged data that is not touched. But the tree data structures are
  often more CPU and memory hungry than some flat data structure like `HashMap`
  or `Vec`, so you may want to measure what results in better performance and
  memory consumption for you. I know of the
  [`rpds`](https://crates.io/crates/rpds) and
  [`im`](https://crates.io/crates/im) crates implementing some persistent data
  structures.

## Alternatives

I've done some minimal benchmarks. I've found none where the `ArcSwap` would
perform worse than `RwLock<Arc<T>>`. Some other benchmarks could find such a
situation and benchmarks are not a real-word load, so it does not have to be a
clear win for `ArcSwap` over some conventional means, even with the extra
guarantees about being lock-free.

Also, the [`crossbeam-utils`](https://crates.io/crates/crossbeam-utils) crate
promises to provide a type with similar capabilities soon, but with slightly
different performance characteristics (but the work in progress seems to
allocate sometimes, which would make it unusable for the signal handlers).
