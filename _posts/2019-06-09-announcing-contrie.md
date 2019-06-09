# Announcing the contrie ‒ concurrent maps and sets

This is partly an [announcement](#whats-in-the-crate) of a new crate folks might
find useful, partly a [call for participation and help](#help-wanted) and partly
a [journal like story](#behind-the-scenes) how the crate came to being. Read on
(or not) or skip to the parts that seem interesting to you.

And I know, I'm bad at naming libraries, the name is silly.

## What's in the crate

The goal of the [`contrie`] crate is to provide a concurrent [lock-free] map and
set, based on a concurrent hash trie data structure. That is, something with
similar interface as [`HashMap`] or [`HashSet`], but allowing multiple threads
to do stuff to it at the same time, without locking.

What good is that? Well, let's say you're writing a chat server ‒ maybe a
new implementation of an IRC daemon. And you want it to be asynchronous and
multi-threaded. You need to keep certain data structures around, for example
a list of currently connected users. Sometimes, you need to send a message to a
particular user (look up in that data structure), sometimes you need to send a
message to all of them (iterate through it), and also people join and leave all
the time. You could certainly use something like `RwLock<HashMap<UserName,
User>>`. But, when a user is being added to the map, other threads get blocked
if they want to iterate or lookup in there. That might not be a big deal, but it
certainly doesn't sound great either. And from certain load that lock becomes a
choke point of performance, no matter how many cores the server has.

So, this crate provides a data structure offering the functionality without any
blocking ‒ you can iterate and lookup and modify from as many threads at once as
you like. There can be some contention in there, but *mostly* only if two
threads try to add or remove the same key.

Under the hood, it is a modified and simplified version of the data structure
from this [Wikipedia entry].

### Maturity

I've just released the `0.1` version on [crates.io]. And I like the philosophy
of „release early, release often“. In other words, this definitely is not
finished and highly polished library yet. I do intend to continue working on it
during my limited spare time though, and I'd like the library to reach that
state *eventually*, not abandon it. But it is also an opportunity for you to
both help with it and influence its further direction of development.

Currently, there are mainly two things available:

* The [`Raw`] type. This one holds the actual implementation of the data
  structure, all the `unsafe` code and similar. The interface is somewhat
  minimal and closer to what is comfortable for the implementation that the use.
  But the aim here is to separate the `unsafe` code from building a reasonable
  API and to reuse the same difficult code between the map and the set (or,
  possibly multiple flavours of these).
* The [`ConMap`] type wraps the [`Raw`] one into *somewhat* more convenient API
  to implement a map. This one holds elements behind [`Arc`]s, to work around
  some technical limitations. There are other possible options, so this is where
  other flavours could come in.

Things that are definitely missing are:

* A set type.
* A feature-gated support for the [`rayon`] crate.
* I'd like the [`Raw`] to be a bit more flexible ‒ under the hood it is a trie,
  but currently it is restricted to being a trie of 64bit hashes. It should be
  possible to extend it to allow arbitrary bit keys, or at least arbitrary
  fixed-sized ones.

Also, I'm not exactly satisfied with the API of [`ConMap`]. I'm certainly better
at implementing data structures than designing their APIs.

## Help wanted

If you search [crates.io] for a concurrent map, you're likely to find several
immature and abandoned ones. I'd really like this one to be different and to
grow into a mature and usable one eventually. If you, too, would find one
useful, I could use some help with it.

So, what would really help:

### Provide feedback

If you have a use case, please try it out if it works for you. If it gives you
pain while using it, I want to know what is cumbersome. If you have ideas on how
to improve it, even better.

Or at least read the [documentation] and think if it *could* work for you or
where you think the pain points would be.

### Code review + try to break it

I've tried to read the code multiple times. I've written comments, arguing why
the unsafe code should in fact be safe. But, well, bugs happen. So, if you enjoy
reading unsafe code, please have a look at it.

Alternatively, try to break the thing. Failing tests (that are correct
themselves) in form of pull requests or bugreports with code snippets are great
:-).

I'm always happy to learn what I did wrong (well, my ego of course prefers
learning what I did right, but I try to learn from mistakes).

### Help writing the code

The crate is split into the hard parts, where unsafe code lives, and the easier
parts, where it is just modeling a nice API around it. Documentation and
embedded code examples are always a good place for improvements.

So, there are opportunities for both easy and hard coding tasks and I'll be
happy to help out, mentor or discuss ideas on all the levels.

It lives in a github [repository]. I haven't created issues for everything
that needs doing yet, though there's at least a file with a [TODO list] in
there.

Eventually, I'll probably get around to creating the issues and try to get them
listed in the Call for Participation in [This Week in Rust].

### Experiment

As mentioned, implementing the data structure is only one thing of what makes a
good library. A nice and convenient API is very much needed too and that will
probably take some experimentation (after years of writing C++ code, my
intuition for what is a good API design suffered a bit ‒ I can differentiate
between several levels of „obviously terrible“, but once I get to „that's
OK-ish“, I no longer see much difference, and I guess there are people who *can*
see the difference between mere „OK-ish“ and „brilliant“). If you have an idea
to try out, don't hesitate. The existence of the [`Raw`] type should provide the
building element, without having to invest in understanding all the details of
the data structure and the unsafe code.

It is fine to add a different flavour of a map or a set and let multiple of them
live in the crate, at least for now. Eventually it would be nice to choose only
the best ones instead of having a dozen of slightly different map types, but
that means having something to choose from.

### How to join in

The project is too small to have some specific rules in that regard. In other
words, you can create issues, send pull requests or reach out through [email],
as you like.

## Behind the scenes

This is the story why the crate came into existence.

Some time ago there was this thread about [not having concurrent maps]. Well,
that one actually talks about why they are not in the standard library,
nevertheless the gist is there are no mature ones outside of it either. And, as
with the [motivation] for [ArcSwap], it felt *wrong* that Java could have some
goodies Rust doesn't. But that didn't trigger any action from me yet.

But then, I've stumbled on this [Wikipedia entry]. That thing looked
interesting. It looked like the kind of challenge I like ‒ a hard to implement
data structure, nevertheless with well defined purpose and boundaries and
relatively small so it's possible to actually implement during occasional free
day or evening (yes, some people have a weird definition of relaxation, but I
just need to stretch the mind to the limits sometimes).

However, Wikipedia contains just enough information to grasp the basic idea of
the data structure and the general properties, but not all the details needed
for actual implementation. Luckily, the original [article] can be downloaded
from the Internet, so I've started to study that one too.

I'm not going into too many details (for them you can study both the [article]
and my source code in the [repository]), but I have to admit that the data
structure described there seemed very complex at first. In particular, there are
4 different kinds of nodes in the trie. Between each neighboring levels of
branching nodes there's a single-pointer walk-through node (which they call
an `I`-node) that helps with avoiding some kinds of race conditions and with
implementing snapshots of the data structure for consistent iteration through
it. And all that complexity is in the context of a lock-free code, which is in
general error prone and very hard to test in all the corner cases.

In other words, I felt like there was no chance to get anywhere near to correct
implementation during the first attempt and that I have to simplify it. Also,
storing the single-pointer nodes felt wrong ‒ that doubles the number of memory
loads to reach the element and allocating a lot of small structures has high
overhead for the memory allocator. But the practical implementability was the
bigger reason.

So, the current code doesn't contain the `I`-nodes. It doesn't offer the feature
of *consistent* iteration ‒ if you iterate through the data structure, the
iteration will reflect *some* changes done during the iteration (which is
probably fine for most use cases ‒ in the IRC example, the message of the day
may or may not be sent to the user that *just* joined during the iteration). The
branching nodes in the article use a bitmap field + only the non-null pointers
(therefore, if the node is only half-full, it allocates only half of the size a
full one would). Mine are always the full size, containing the null pointers.
In return for that, I *hope* I got a slightly faster implementation (to be sure
I'd have to write the other version too, because comparing it to a Java
implementation wouldn't really work). But more importantly, I have much more
confidence that what I wrote is actually sane.

To do that, I had to rethink and re-design some of the corner-case details.
For example, the process of pruning dead branches after removal of element is
somewhat different. Their approach however provided the inspiration ‒ I do it
differently, but in the general direction of their idea.

And then I had to actually spend a lot of the free evenings, adding
functionality piece by piece, writing tests, trying to argue correctness and
figuring out how to fit all that into Rust ownership model ([`crossbeam-epoch`]
helps a lot with managing the memory, but not with wrapping it behind an API
wall).

I'm not ruling out trying to implement the original version one day. It might be
another level of the challenge and being able to compare them would be useful.
But not today. However, after implementing the simpler version I feel like I
finally understand what they do in the article.

## The future

It's always hard to make predictions. But as I said above, with some help it
should be possible to bring this up to a usable shape. The split between the
core algorithm and API layers should make researching suitable interface easier.

I'm not sure if this should stay a separate crate in the long term ‒ it might be
easier to find it if I eventually „donated“ it to the [`crossbeam`] project. But
I have no idea if the project would like such donations and I feel like at this
immature state, the crate is better on its own.

All that depends on if someone actually starts using it or if it goes down as
another exercise project.

[`contrie`]: https://crates.io/crates/contrie
[lock-free]: https://en.wikipedia.org/wiki/Non-blocking_algorithm#Lock-freedom
[`HashMap`]: https://doc.rust-lang.org/std/collections/struct.HashMap.html
[`HashSet`]: https://doc.rust-lang.org/std/collections/struct.HashSet.html
[crates.io]: https://crates.io/
[`Raw`]: https://docs.rs/contrie/0.1.0/contrie/raw/struct.Raw.html
[`ConMap`]: https://docs.rs/contrie/0.1.0/contrie/map/struct.ConMap.html
[`rayon`]: https://crates.io/crates/rayon
[repository]: https://github.com/vorner/contrie/
[TODO list]: https://github.com/vorner/contrie/blob/master/TODO
[documentation]: https://docs.rs/contrie/0.1.0/contrie/index.html
[This Week in Rust]: https://this-week-in-rust.org/
[not having concurrent maps]: https://users.rust-lang.org/t/why-are-there-no-concurrent-collections-in-std/26934
[motivation]: https://vorner.github.io/2018/06/24/arc-more-atomic.html
[ArcSwap]: https://crates.io/crates/arc-swap
[wikipedia entry]: https://en.wikipedia.org/wiki/Ctrie
[article]: https://www.researchgate.net/publication/221643801_Concurrent_Tries_with_Efficient_Non-Blocking_Snapshots
[`crossbeam-epoch`]: https://crates.io/crates/crossbeam-epoch
[`crossbeam`]: https://github.com/crossbeam-rs/
[`Arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html
[email]: mailto:vorner@vorner.cz
