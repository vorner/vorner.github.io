# Two holiday crates

Due to various factors, I've ended with more free time these Christmas holidays
than usual. A large part of that went (and still goes) into some Rust coding.
And maintaining existing stuff is fun only so far, so I've somehow happened to
start two new crates (so I have more things to maintain in the future, yes, I
see the problem).

In case someone else also finds them useful, I'm going to introduce them here,
with the motivation why I wrote them.

## The [`adaptive-barrier`]

Some time ago, I was pointing out that Rust contains [many a trap around
panics][dont-panic]. Since that time, I've had to work around these traps in
various places. One that was still problematic was using [barriers].

Now, one doesn't need to use these very often. Certainly less often than mutexes
or channels. Part of that reason is probably the existence of [rayon], because
it handles all kinds of map-reduce scenarios very well. But there are times when
they are useful. I use them in some unit tests when checking synchronization of
things. Due to the randomness of multiple threads interacting with each other,
I want to run the same test multiple times. But starting new threads for each
test is slow, so I use barriers to reuse the threads but still run the same
interaction in lockstep.

```rust
use crossbeam_utils::thread;

#[test]
fn test_producer_consumer() {
    let barrier = Barrier::new(2);
    thread::scope(|s| {
        s.spawn(|_| {
            for i in 0..ITERATIONS {
                barrier.wait();
                produce(i);
            }
        });

        s.spawn(|_| {
            for i in 0..ITERATIONS {
                barrier.wait();
                assert_eq!(i, consume());
            }
        });
    });
}
```

What happens if the test fails with a panic? That one thread will get torn down.
But the test will still wait for the other thread to finish too and that will
never finish, because it now waits on the barrier ‒ for the dead thread that's
no longer there. And getting stuck forever is not the best way for a test to
fail ‒ github actions will time out eventually, but that takes hours. It's both
unfriendly to contributors (they get the failure after a long time) and to
github, because it takes up their resources unnecessarily.

Spraying the code with kinds of „check that the other thread still lives“ helps
only a little, because it is inherently racy ‒ the other thread might die in
between the check and the wait.

### Building the check into the barrier

The solution I've come up with is building my own [`Barrier`]. This one doesn't
get the number of threads to synchronize in its construction. Instead, it can
track how many threads it should expect at runtime.

How do I do that? The `wait` on the standard library takes `&self`, so consumers
stuff the barrier into an `Arc` or share it between the threads in some similar
way. My barrier takes `&mut self`, but can be cloned. Each thread gets its own
private clone. And the barrier can count how many clones there are of it.

This means one can add or remove the clones as the time goes and the barrier
will notice. And if a thread panics, it'll destroy its clone as it unwinds, and
the other threads don't need to wait for it any more.

Actually, the destructor can detect if it is being called during a panic unwind
or just during normal destruction. The consumer chooses how it deals with panics
it observes this way ‒ either by just removing the clone (and continuing in the
other threads), or by poisoning the barrier (which will then panic from all
calls to `wait`, tearing down the whole algorithm or test).

```rust
use adaptive_barrier::{Barrier, PanicMode};
use crossbeam_utils::thread;

#[test]
fn test_producer_consumer() {
    let mut barrier = Barrier::new(PanicMode::Poison);
    thread::scope(|s| {
        s.spawn({
            let mut barrier = barrier.clone();
            move |_| {
                for i in 0..ITERATIONS {
                    barrier.wait();
                    produce(i);
                }
            }
        });

        s.spawn(move |_| {
            for i in 0..ITERATIONS {
                barrier.wait();
                assert_eq!(i, consume());
            }
        });
    });
}
```

### The implementation

It is heavily inspired with the `Barrier` inside the standard library ‒ a
`Mutex`, `CondVar` and few counters. The difference is, an `Arc` is placed
inside it (so the cloning is really cheap) and some manual additional counting
is done in the `Clone` and `Drop` implementations.

As it is such a simple thing, it has no dependencies (except for `std` itself)
and compiles even on some very old Rust (I have 1.26.0 in the CI).

### The future plans

I feel like there's nothing to be discovered about the API. It has the `wait`
method, just like the implementation in standard library, and that's it.

So I'll probably let it live for a while and then call it „stable“.

## Saving memory with [`squash`]

Let's say we want to keep a lot of smallish strings around. We could use
something like `Vec<String>`.

If our string is maybe 30 bytes long, it'll allocate 30 bytes on the heap and
will take up 24 bytes inside that `Vec` (we assume we are on a 64bit
architecture). That's 54 bytes (and some allocator overhead). That also assumes
there's no unused capacity ready for the string to grow, but we can always call
`.shring_to_fit` before putting it together with the few million other strings
we already keep around.

Let's say we want to save some space. What can we do?

We can use `Box<str>` instead of `String`. That takes only 16 bytes inside that
`Vec`, so an improvement, and it never stores any extra capacity.

One can also use something like the [`smallstr`] crate. That one will store
small strings "inline", without another heap allocation. But we have to choose
how large strings fit inline ‒ and we either waste space by making the inline
bigger than needed for really short strings, or by not saving anything in case
they don't fit.

This crate allows for storing the string with 31 bytes on the heap and 8 bytes
inline in the `Vec`. That's 39 bytes instead of that original 54 (or 46 in case
of `Box<str>`). Similar to the owned string slice, it can't change its length
any more, so it's better for long-term storage in large quantities than for
manipulation.

The library supports similar storage of arbitrary owned slices.

### How it works

In Rust, slices (and string slices) use fat pointers. That is, there's a pointer
to the start and a length. That length is `usize` ‒ 8 bytes large on 64bit
machine, even if 7 of these bytes are zeroes.

This library puts the length on the heap, just before the actual data. That
allows us to store only a thin pointer to it (that 8 bytes inline). The length
is encoded in variable number of bytes, depending on the actual value ‒ so
smaller strings will use smaller number of bytes for the length. There are some
limits on the size that can be encoded, but it is in the orders of „Huge“ (the
default encoding has a limit of 2^38, which is 256GB ‒ by that time you're no
longer in the land of storing a lot of smallish strings any more).

### Yes, but why?

You might be asking if this is worth the effort, that the savings are not that
great and one might not want the unsafe code needed to implement it around (yes,
this needed some unsafe code, though I'm quite confident about it and [`miri`]
seems to agree). And you might be right, it is easy to get burned by unsafe code
(*especially* innocent easy-looking unsafe code).

The thing is, this is the very first version of the library. It hasn't grown to
its full potential yet and I have some futures in mind that'll be helpful for
few practical use cases.

First, I want to include a limited reference counting (as an opt-in, at the cost
of needing additional byte of the length sooner). That, in turn, might come
handy when implementing persistent data structures. When designing a persistent
data structure, one usually ends up with using `Arc`s and some kind of B-tree,
possibly with nodes being heap-allocated owned slices of pointers to more nodes.
With `Arc`, one uses a fat pointer of 16 bytes + 16 bytes on the heap of the
reference counts. With this, one could cut it down to something like 8+2 (though
alignment of the pointers might introduce other overhead in the form of padding;
I wonder if some kind of packed storage of pointers would be helpful here and
what the speed cost would be).

The other feature would be to somehow (I'm not sure about the API yet, only
about the memory layout) encode *multiple* strings/slices in the same memory
block. The biggest overhead right now is that 8-byte long pointer (and the
allocator overhead, which is unknown). If you are having a phone book with a
name, surname and an address (three shortish strings), one could encode them
with 8 (pointer) + 3 (lengths) + 1 * allocator, instead of the usual 3 * 16 +
3 * allocator for the `Box<str>` encoding. That starts to feel like a
significant improvement (and it might or might not be faster, due to being
together in cache).

### The stability status and such

Unlike the above [`adaptive-barrier`], this is very much in the experimental
stage. The API is quirky at places, many things are missing (there's a
laundry-list of TODO notes inside the `lib.rs`). But it might be possible to get
some memory savings from it already, if you are in the need.

I'm not sure when or if I'll get around to implementing some of the features, or
if there's even a possibility of some nice API.

## What's next

As with everything, I don't really know how useful these thins will be in
practice. I'll be switching between other crates too, and will return to these
depending on how much I personally need them and how much interest there seems
to be in them from others.

If you find them useful or find that something from the TODO list would be a
good exercise you'd enjoy, feel free to join in on the github repositories.

[barriers]: https://doc.rust-lang.org/stable/std/sync/struct.Barrier.html
[rayon]: https://crates.io/crates/rayon
[`Barrier`]: https://docs.rs/adaptive-barrier/0.1.0/adaptive_barrier/struct.Barrier.html
[`smallstr`]: https://crates.io/crates/smallstr
[`adaptive-barrier`]: https://crates.io/crates/adaptive-barrier
[`squash`]: https://crates.io/crates/squash
[`miri`]: https://github.com/rust-lang/miri
[dont-panic]: {% post_url 2018-07-22-dont_panic %}
