# The Nuclear Reactor Design Pattern

If I'm going to invent another name to something that already exists, then I
apologise ‒ I simply don't know about the other name. If you do, I'm all ears.
Or maybe it just seems obvious to you.

Anyway, there are two kinds of design patterns ‒ low-level ones and high-level
ones. A low-level one is for example the Factory design pattern. It tells you
how to solve some small detail. The high-level ones are more about the design of
the whole application and about philosophy. They are also a bit less specific.

I'm going to talk about one of the latter ones, one I call the Nuclear Reactor.
I've been using it for much longer than I have a name for it or even decided it
is a form of a design pattern. It is in no way *specific* to Rust ‒ I've used it
in C, C++ and Perl and other languages too. But I feel like Rust somehow
slightly encourages people towards its use, or is more popular in Rust for some
other reason.

## A Pasta sidetrack

Everyone has heard about the dreaded Spaghetti Code. It's full of `goto`s and
jumps and exceptions and it's quite hard to follow in which order and how the
control flows through the code. Imagine you're staring at a plate full of
spaghetti and want to know where this particular one goes or if these two ends
are connected somewhere in the middle of the stuff. Of course, without taking
a fork and pulling it out. Some C or BASIC code is famous for this. Note that
a `goto` doesn't *necessarily* signify a spaghetti code, there are uses of
`goto` that make the code *more* readable and idiomatic.

Then we have the Lasagne Code. This is the rule of the „All problems in computer
science can be solved by another level of indirection“ taken to the extreme.
Everything is full of proxies and accessors and virtual somethings. And after
examining 17 levels of function calls and classes, you no longer know what
you've been looking for in the first place. Or even worse, you reach the bottom
and it seems the functionality still isn't there. Just levels, none of them
doing the actual thing.

I've recently discovered another one, let's call it the [Bulgur] Code (is bulgur
still pasta?). It's when you take the rule that no function should be too long
into the extreme. When you shred your code into tiny 2-line functions, that do
nothing more than just call another 2-line functions. This, however, differs
from the Lasagne Code above ‒ there are separate layers and each one adds some
kind of indirection. This *doesn't*, it simply goes to extract every single
block of code into its own function (or to multiple, just to make sure). Instead
of being able to read the whole logic as one function, you have to jump through
the file(s) and *assemble* the logic in your head. It's all just tiny bits off
*stuff* and no actual meat to bite into.

It is probably obvious that I don't fancy working with either style of these
Pasta Code. But what do they share in common?

They try to pretend that by splitting a hard problem into smaller bits, it stops
being a problem (maybe with the exception of the Spaghetti one, that just seem
to *happen*). That is not true. While each bit is simple *in itself*,
nevertheless the whole thing is still complex. Except, it's no longer in one
well defined place to look at. The complexity is spread though everything. It's
like Guerilla complexity, no concrete place to attack it and eliminating or
understanding one bit doesn't really help with the whole.

## The principles of Nuclear Reactor

Assume we have a hard problem to solve as part of the application. Well, that
sucks! (or maybe, that's great!, because there's some *interesting* part of the
code to work on). Maybe an event loop than needs to be fast. Or complex data
structure that keeps the data compressed so it eats less RAM.

So we'll do the exact opposite of the above. We take the hard problem and that
problem *alone* and put a strong API wall around it. This creates a stark
contrast of complexity between the hard core inside and the *everything else*
around it.

The idea behind this is, the problem is still hard, but when working with it, it
is all in one and relatively *small* area. The whole hard part can be studied
and thought about as a whole, which is not possible if it is spread through the
rest of the application. And it also tends to be well-defined what it should do,
it has strong sense of purpose.

Locking the problem away like this has some more advantages. If the API is well
defined, then it is easy to reason about it, both from inside and outside, on
intuitive level. And it is easy to test.

As with the real nuclear reactor, it's quite a good idea to never breach this
kind of separation. Like, if someone comes with *a briliant idea* to put a bit
of unrelated business logic inside the API wall because it is easier to
implement, or that some of the hard stuff can be passed onto the caller. The
danger is that the complexity will spill and entangle with the rest, making the
reasoning about the problem much harder.

Similarly, if someone comes with a requirement that the core should have
exceptions around its behaviour that doesn't logically follow from the
definition of the problem, they should *not* go inside. Like, even if it makes
sense for your application to consider an disjunction of empty set of terms to
evaluate to false, it needs to be done outside of the logic core, because that
just makes no sense in any sane logic.

The thing inside should do just that one single problem. Be as dumb as possible.
Yes, fast, elegant, but *dumb* in the sense it is easy to predict what it should
do. Smart things have the ability to surprise and make their own damn minds.
Leave all the smartness on the outside.

## How does this relate to Rust?

As I mentioned above, the principles of this design pattern can be used in
whatever language. It probably doesn't even have to be able to enforce the API
boundary, if programmers are disciplined enough to not violate a boundary
written only in documentation.

However, it seems to be more common in Rust than what I've seen in other
languages. I think there are several reasons for this.

First, Rust attracts people who like hard problems. So there are lots of crates
that solve hard problems. And when the problem is solved, it usually goes into a
crate of its own, with *some* API. The usage is less hard. Furthermore, even
internally, the difficulty contrast is used around these hard problems ‒ for
example [rayon] has a lot of API surface. The API itself is not easy to create,
but there's this even harder core of scheduling and synchronizing tasks. This
actually lives in its own `rayon-core` crate. You as the rayon's user are not
expected to come into contact with it, it's mostly for the authors of rayon
itself (well, there are also other reasons for this particular split).

Second, a psychological influence. The teaching materials are very explicit
about the importance of separating `unsafe`ty behind an API wall. Even if your
hard problem isn't necessarily `unsafe` in the Rust sense, the notion of keeping
the dragons locked at one place feels natural in that context.

The third reason is Rust's type system. It allows *building* the barrier for the
difficulty without actually getting into the way when using it. Unfortunately, I
can't say why that is or what part of the type system does that. But in the
other languages I've used, the API either leaked too much or was getting in the
way too much.

## Is this just a fancy name for encapsulation?

Not really. Encapsulation is a requirement for the Nuclear Reactor. Not every
encapsulated thing is, however, the Nuclear Reactor. For that, there needs to be
some significant difference between the difficulty inside and outside and the
strong stance of never leaking the hard stuff outside and never allowing any
irregularities inside. For example, a module to connect to the server and
subscribe to some kind of updates can be encapsulated, but it is often „just“
bunch of code around downloading things, nothing particularly hard. The code
using it will be about the same difficulty.

It's as much about the philosophical stance as it is about the code itself.

## When to use it

I've never regretted using this pattern when I managed to use it.

However, I don't claim to forcefully hammer it into every application. For it to
work well, there needs to be some *well-defined* and *extractable* problem to
solve.

Though it doesn't have to be the hardest part of the application, or the only
use of the pattern in the application. The pattern can live side by side in
several instances. Interestingly, it can also *nest* (yes, the [rayon] example,
or I do something like that in [arc-swap] ‒ the [`Cache`] is not easy to prove
correct, but much easier than the [`ArcSwapAny`] itself).

[`BuildHasher`]: https://doc.rust-lang.org/std/hash/trait.BuildHasher.html
[Bulgur]: https://en.wikipedia.org/wiki/Bulgur
[rayon]: https://crates.io/crates/rayon
[arc-swap]: https://crates.io/crates/arc-swap
[`Cache`]: https://docs.rs/arc-swap/0.3.11/arc_swap/cache/struct.Cache.html
[`ArcSwapAny`]: https://docs.rs/arc-swap/0.3.11/arc_swap/struct.ArcSwapAny.html
