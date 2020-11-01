# Stabilizing `arc-swap`

For some time, I've intended [`arc-swap`] to go to 1.0 version and stabilize. It
took longer than I've expected, for various reasons, but I believe it's finally
there.

TLDR: I've released an [RC1]. Please test and report anything as an issue to the
[repository], especially things that can't be changed once this really goes 1.0.
I'll wait few weeks and then, depending on what gets reported, release a 1.0 (or
RC2 if there's a lots to deal with).

## If you don't know arc-swap

It's a synchronization primitive to store and access an `Arc`, concurrently.
It's optimized for situations with frequent reads and rare updates. It is
[lock-free] for readers (on all accesses except the first one on each thread it
is actually even [wait-free]) and won't suffer from contention even with many
concurrent threads accessing it at the same time (unlike eg. `RwLock<Arc<T>>`,
where the CPU cores will fight over the atomic in the lock), provided few rules
are followed.

The rest of the library contains bunch of auxiliary types to get the best of
this thing in various use cases (caching, projections into data, …).

Sure, it's somewhat niche thing and the chances are you don't really need it in
your use code. But in some applications it might make latencies more stable.

There are some past articles about what is under the hood, so if you're
interested in these kinds of things, feel free to browse this
[blog](/index.html).

## Changes since the previous versions

From a high level, the API stays very similar. Unless you do something weird or
special, you should not need to change anything in your code.

Parts of the API got removed:

* The `load_signal_safe` method. It was making the API and implementation hairy
  and I suspect I was the only user of it anyway, so I've extracted the specific
  functionality directly into [`signal-hook-registry`]. That also means that
  [`signal-hook-registry`] now depends only on `libc`.
* The `rcu_unwrap` method. The whole concept of that method is a trap for the
  unwary and should never have existed in the first place. If you know enough to
  use it correctly, you also know enough to write the required 3 lines of code
  that was its implementation.

A lot of internal traits was made private or sealed. I want to stabilize the
public API, but not private things.

There was a way to slightly tweak the algorithm used for the protection of the
reference counts. This was done by specifying a second type argument to the
[`ArcSwapAny`]. This is still possible, but the abstraction moved to a higher
level and it allows swapping out most of the algorithm being used, not just one
tiny implementation detail. But as this also became a sealed trait without any
exposed methods, the details don't matter for users. The difference is the set
of types the crate provides to choose from (which is a bit limited at the
moment). But also, it is no longer possible to implement them right now by
users.


And now there's an explicitly stated minimal compiler version policy and a bit
of guide-level documentation
[embedded](https://docs.rs/arc-swap/1.0.0-rc1/arc_swap/docs/index.html) in the
common API documentation.

## Future plans

After proper stabilization, I'd like to play a little bit with additional
locking strategies (the above mentioned type argument). It turns out there are
some other ways how to solve the problem of updating the reference counts in the
`Arc` in a safe way when not locking it. They bring different trade offs.

For example it should be possible to get significantly faster reads (as in, in
theory it should be possible to get to speed equivalent to normal access of
ordinary `Arc` in a local variable). But it needs non-trivial OS support and
possibly can delay the destruction of previous versions of data. That would mean
one can't store anything too large there and that it needs to be `'static`
(owned types). So eventually there could be some more strategies to choose
from, depending on what the user can afford.

Eventually, I'd like to make the `Strategy` trait implementable by the user too.
But that'll only happen once I have some confidence that the trait is right.
Currently, there's only one real implementation of the trait and I might need to
change it for the new ones.

## Call for action

As stated above, please test. I don't expect many big problems with the API ‒
the API stayed mostly the same for a long time already. But I still could have
overlooked something and more people having a look at it, using it or even
reading the documentation will help.

If you already use arc-swap, please try updating (you don't have to put the RC1
to production, but if you can try it out and run tests, I'd be happy).

[`arc-swap`]: https://docs.rs/arc-swap
[repository]: https://github.com/vorner/arc-swap
[RC1]: https://crates.io/crates/arc-swap/1.0.0-rc1
[`signal-hook-registry`]: https://crates.io/crates/signal-hook-registry
[`ArcSwapAny`]: https://docs.rs/arc-swap/1.0.0-rc1/arc_swap/struct.ArcSwapAny.html
[lock-free]: https://en.wikipedia.org/wiki/Non-blocking_algorithm#Lock-freedom
[wait-free]: https://en.wikipedia.org/wiki/Non-blocking_algorithm#Lock-freedom
