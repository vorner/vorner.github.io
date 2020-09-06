# If you want performance, cheat!

Have you ever been banging your head against a performance problem (or percieved
performance problem, the kind of „this really should be possible to make faster“
kind), running the profiler, tweaking, shaving off milliseconds, only to
eventually discover (or be advised by someone) a way to completely sidestep it
by some kind of neat cheating trick?

Sometimes, I feel like there are plenty of these in our industry and each one is
nice to know or at least in a way interesting. So I'm going to share two such
cheats I've lying around in my code. You can share your favourites too.

## Atomic Arcs and caching

I've written the [arc-swap] crate (and I'm slowly working towards a 1.0 release,
but there's always something else to work on too so it takes time). The idea of
the library is that there's a place where an `Arc` can be stored. This place can
be shared between multiple threads and the threads can read the `Arc` or change
it to point somewhere else. Somewhat like `RwLock<Arc<T>>`, but faster and
without the locking.

This can be useful in situations where there's some global-ish data structure
that is very often read but seldom updated. Thing about configuration that can
be updated by the user but is used all the time, or a routing table that is used
to route each packet, but might need to be changed upon an occasion.

There are a lot of little ugly technical details inside that, mostly coming from
the fact that if you don't want to lock, you have hard time making sure the
writing thread doesn't decrement the reference count to 0 and drop the value
just before another thread tries to increment it and claim shared ownership.
Doing this safely, without any locking at all and fast at the same time is still
a topic of academic research.

So while [arc-swap] is faster than `RwLock` and suffers much less from
contention between the reader threads (there's almost no penalty from arbitrary
number of threads trying to read it at the same time), I'd like to make it *even
faster*. The hot path still contains some atomic operations, one of them in the
[SeqCst] ordering. This is slow (relatively speaking). And while I'm trying to
keep an eye on what results of the research could be incorporated, this is
unlikely to get an order of magnitude speedup.

That speedup came when someone on Reddit recommended a nice trick (sorry, I
don't remember their name ☹). Most of the time the value read from there is the
same as the one read last time. So each thread keeps its own cached copy of
fully loaded private `Arc`. Instead of *loading* it from the shared storage each
item, it only *revalidates* the cached value.

Instead of the whole synchronizing dance, it simply loads the value of the raw
pointer from the shared storage with a [Relaxed] ordering. That one is *fast* ‒
depending on the platform, it is either just the very same load as an
unsynchronized read from a variable (it might disable some compiler
optimizations, but doesn't incur expensive memory barriers or locked
instructions) or something very close to that. It doesn't synchronize the
pointed-to value in any way and doesn't protect the reference count. But that
doesn't matter, because we use the pointer only to check that it is still the
same as the one we have. If yes, we are fine using the value we already have.
Only if it is different (which is the rare case) we do the whole loading
operation and store the value for later.

And this does bring the desired order of magnitude speed up. It comes with its
own set of disadvantages, like having to keep the [`Cache`] around, so it's
suitable for singleton variables, but not for interlinked data structures, and
the old value's drop might get delayed. Therefore, this is an add-on on top of
the basic [`ArcSwap`], instead of replacement.

The speedup probably doesn't matter in practice. Even the speed of the full
loading is going to be dominated by the actual *use* of the data. But all that
is certainly *cool* and who wouldn't want their hot path to be guaranteed not to
wait on any kind of lock whatsoever. It also sidesteps the debate if it's OK for
async application to use the usual blocking locks if they are locked for a short
time, because this just doesn't lock at all.

## Bumpallo herds

The above is not a new thing. But the following one is. But a little bit of
backstory first.

Let's say we want to write an algorithm that needs to allocate a lot of small
things, assemble some kind of data structure out of them. Then it plays with it
for a while and eventually throws it all away. One such example could be parsing
an input into an AST. The usual way is to just put each node into a [Box]. But
that is slow, because allocating each means talking to the global allocator.
Furthermore, getting rid of the data structure means walking through all the
nodes and talking to the global allocator again, for each freed one. Don't get
me wrong, I don't mean to suggest the authors of the allocator were sloppy. That
piece of code is probably one of the most optimized things around. It is slow
simply because its goal is complex.

Doing flat data structures (using long `Vec`s instead of linked lists or having
fields inline in bigger structs) is one of the ways Rust accomplishes its speed.
But recursive ASTs are not really that well suited for that and that's where we
can use an arena allocator (that's how Rust folks call the thing), bump
allocator or a memory pool (that's how Apache calls it). The idea is that it
allocates a big block of memory from the global allocator. Then, when the
application needs a small chunk, it bites it off that big block, only moving a
pointer in the process.  There's no per-allocation bookkeeping to do, it's super
fast and as a bonus, the memory is close together, having better cache locality.
And when it's time to get rid of the whole data structure, the big block is
returned to the global allocator all at once, without the need to traverse all
the links in there. The downside is, there's no way to deallocate single
allocations, only everything at once (technically, it can be extended to support
stages, deallocating the newest bunch of allocations and returning to an older
state, but that's not relevant to our story).

The one I'm fond of is called [bumpalo]. It needs only [11 fast instructions] on
the fast path to allocate. This gives us an order of magnitude speed up in a
very non-scientific and synthetic benchmark (but it can really speed up your
program if it's allocation bound, for example `rustc` uses a similar thing at
various places). This is a nice trick in itself, but not the one I want to write
about.

Besides limited ways of deallocations, there's another downside. The allocator
is not thread safe. It really *can't* be made thread safe and still stay this
fast, because even a single [`fetch_add`] will be a significant slow down
compared to that 11 non-atomic instructions, and we would likely need more than
that [`fetch_add`].

So we gained fast allocations, but lost the ability to parallelize our workload
easily with [rayon] or crossbeam's [scoped threads]. If wanted to do something
like this (let's assume that the `usize`s are actually some complex nodes in our
AST and we link them together):

```rust
let arena = Bumpallo::new();
let data: Vec<&usize> = (0..10_000_000)
    .into_par_iter()
    .map(|n| arena.alloc(n))
    .collect();
dbg!(data);
```

The worker threads would be fighting over the one arena, which would be
incorrect (data races on the allocation pointer and such) and slow on top of
that (fighting over the cache line where it lives). Rust is right in refusing to
compile this cursed code.

Rayon has the [`map_init`] method that lets us create a separate instance for
each thread.

```rust
let arena = Bumpallo::new();
let data: Vec<&usize> = (0..10_000_000)
    .into_par_iter()
    .map_init(Bumpallo::new, |arena, n| arena.alloc(n))
    .collect();
dbg!(data);
```

But that doesn't work either, because the arenas with the memory blocks in
them are dropped once the rayon task terminates and the references become
dangling. Dang!

This „choose only one of arenas or threads“ was frustrating me long enough so
I've decided to go and attack it headfirst. And failed at the first several
attempts.

### Sharding

Obviously, using a `Mutex<Bumpallo>` is not the right way, because the threads
will contend on the mutex and make it an order magnitude *slower* than using the
global allocator (yes, I've tried).

A textbook way to overcome lock contention is to just have more of them and
lower the chance of two threads meeting on the same one. We don't really care
*which* arena the chunk of memory comes from.

So let's have a data structure like this:

```rust
struct Herd {
    instances: Vec<Mutex<Bump>>,
}
```

During each allocation, it would pick one of them in some way and hoping it
would be free. If not, it would try the next one (in circular manner). To lower
chance of collisions in future, it would store the index in a thread local
variable and start at that one next time. The hope is, the threads would
eventually settle each on their own unique index and their mutex would always be
free, which is well known to be quite fast. Only when threads get rescheduled
between CPUs or such would they have to shuffle again.

While the first assumption about the threads getting an unique index fast enough
worked well, it still turned out to be slow. Any kind of read-write atomic
operation (like [`fetch_add`] or [`compare_exchange`]) will ruin our great speed
of *fast* 11 instructions, remember? And, what would you guess is hidden inside
the `Mutex` thing? Even implementing a bare-bones locking without all the mutex
bells and whistles (like thread parking, as we simply move onto the next lock)
won't help. So, are we completely out of luck?

### Rescuing the thread instances

We can't afford to put the arena into a thread local storage (yes, that is slow
too!), but the thread can certainly keep it in a local variable. The problem is
that when the thread (or rayon task) terminates, the arena either needs to be
dropped (which frees our precious data structure) or moved somewhere else (which
ends the lifetimes for which we can have the memory borrowed, according to Rust
borrow checker rules).

But there are ways to make sure some memory or references are not really
*invalidated in practice*. If we put something into a [Box], we can move the box
around as we like, the heap allocation will still stay at one place. We could
guess that the `Bump` thing contains something like a `Box` or `Vec` inside
too, to hold the actual memory block, but let's not take any risks for now.

So, if we take the `Box<Bump>` thing and move it out of the thread or task
when it ends, all the stuff we allocated from it is technically still valid,
only the borrow checker is too limited to see that. Therefore, if we sprinkle
our algorithm with enough `unsafe`, it would be fine.

I don't know about you, but I don't like to see `unsafe` intertwined with the
business logic or complex algorithms. I can do complex algorithms. I can do
`unsafe`. But please don't make me do both *at once*. So what I did is I locked
that really small bit of `unsafe` away.

I have a `Herd` ([bumpalos][bumpalo] are herd animals, right?). This one can
lend out an arena ‒ a small wrapper around the `Box<Bump>`. Once that wrapper
gets dropped, the boxed bumpalo gets returned back to the herd. This way the
memory block lives on, at the same address, so our references can be tied to the
`Herd`, not to the lifetime of the small arenas.

And because each thread has its own independent arena for its lifetime, we
sidestep the whole slow synchronization issue and are still at these really
great 11 instructions (assuming `rustc` is able to see through the box and
optimize the repeated indirection out into a single one at the beginning).

If you want to have a look, I have a [proof of concept]. There's a lot of code
to be written, documented, tested and proven correct ‒ I have only one of the
many allocation methods for now for trying it out. But I intend on finishing it
and publishing it one way or another eventually (currently I'm waiting on an
answer from the bumpalo maintainers if they'd like it in their crate or if I
should keep it as a separate crate myself).

And here are results of the unscientific benchmark on my computer (some oldish
8-core AMD buldozer thing, your mileage will vary). The benchmarks construct
some long singly linked lists:

```
test alloc_directly  ... bench:  58,620,380 ns/iter (+/- 3,793,463)
test alloc_multi     ... bench:  20,452,760 ns/iter (+/- 3,603,708)
test herd_multi      ... bench:     938,045 ns/iter (+/- 533,165)
test herd_single     ... bench:   3,284,031 ns/iter (+/- 395,678)
test locked          ... bench: 318,254,662 ns/iter (+/- 13,216,651)
test single_threaded ... bench:   3,492,762 ns/iter (+/- 236,510)
```

The `alloc` ones are directly using `Box`. The `locked` one is `Mutex<Bump>`,
`single_threaded` is vanilla `Bump` on one thread. The `herd` ones are the
thing described above (both in single and multiple threads).

## Other neat performance tricks

As I mentioned above, there's a lot of these little tricks around that can
really help out. For example, 3D graphics is full of them (Have you ever tried
to render a sphere with only 8 triangles? Or with one square and shaders?). I'll
be happy to hear your favourites. Computers are not getting faster that much any
more, but clever code that can be reused could help make the software to run
faster.

[arc-swap]: https://crates.io/crates/arc-swap
[SeqCst]: https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html#variant.SeqCst
[Relaxed]: https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html#variant.Relaxed
[Box]: https://doc.rust-lang.org/std/boxed/struct.Box.html
[bumpalo]: https://crates.io/crates/bumpalo
[11 fast instructions]: https://fitzgeraldnick.com/2019/11/01/always-bump-downwards.html
[rayon]: https://crates.io/crates/rayon
[scoped threads]: https://docs.rs/crossbeam-utils/0.7.2/crossbeam_utils/thread/index.html
[`fetch_add`]: https://doc.rust-lang.org/std/sync/atomic/struct.AtomicUsize.html#method.fetch_add
[`map_init`]: https://docs.rs/rayon/1.4.0/rayon/iter/trait.ParallelIterator.html#method.map_init
[proof of concept]: https://github.com/vorner/bumpalo-herd/blob/20edeb509bd586d94992589177fcb46cadfa6bcc/src/lib.rs
[`Cache`]: https://docs.rs/arc-swap/0.4.7/arc_swap/cache/struct.Cache.html
[`ArcSwap`]: https://docs.rs/arc-swap/0.4.7/arc_swap/struct.ArcSwapAny.html
[`compare_exchange`]: https://doc.rust-lang.org/std/sync/atomic/struct.AtomicUsize.html#method.compare_exchange
