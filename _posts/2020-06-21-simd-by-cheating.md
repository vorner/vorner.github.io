# SIMD by cheating

Since the last post about [SIMD library plans], I've been experimenting.
Needless to say, it turned out a bit different than originally planned, but I've
something I'd like to share. Maybe it'll be useful for someone or maybe it'll at
least spark some more progress in the area.

If you don't care about the chatter and just want to use it, it's called
[`slipstream`] and is available on [crates.io](https://crates.io). It's an early
release and will need some more work, but it can be experimented with (it has
documentation and probably won't eat any kittens when used). If you want to help
out, scroll down to see what needs help (or decide on your own what part needs
improving 😇).

## Issues with the [original plan][SIMD library plans]

The original idea was to build types that would look like `u32x16<Avx2>`. These
would have operators and methods implemented using the intrinsics from the
[`core::arch`] module, so they would be fast. The user would write the
performance critical code as a function generic over the instruction set and
some kind of helper would choose the best instruction set and call the right
instantiation of the function.

However, this has several drawbacks:

* There are several base types (`u32`, `u64`, `f16`, ...). There are several
  sizes of vectors one would like to have. And there are several instruction
  sets. There's *a lot* of combinations of these and most of them need to have
  different implementation ‒ different intrinsics for different base types, for
  different instruction sets… The naming conventions would allow one to generate
  all these implementations somehow (I've tried generating them by the
  `build.rs`), but this blows up the compilation time. At one point in time,
  the crate was building for about *10 minutes* (while not covering all the
  instruction sets yet). I've managed to restructure it enough to get down to
  about a minute, but it still felt too much.
* There's a catch to using the intrinsics. One can use eg. an `AVX2`-level
  intrinsic in whatever function and it'll get inserted in there alright. But
  unless the function it gets inserted into is allowed to use the `AVX2`
  instruction set too (either by compile time switches or by the
  [`target_feature`] attribute), the intrinsic can't get inlined. That means the
  CPU will load let's say 8 floats into an AVX2 register, then it'll call a
  function containing a single AVX2 instruction to eg. add the 8 floats to
  another register and then it'll return from that function. While the single
  instruction might be *super fast*, the overhead of calling a separate function
  for each such very fast instruction is huge and it just doesn't work
  performance-wise.  So, while using the intrinsics (either directly or in the
  form of wrapping types) one still needs to put the attribute onto the
  function. That makes the approach of using traits/generics unusable for this.
* The code that was produced by using a „generic“ instruction set (one that
  doesn't have any explicit intrinsics, where it just emulates the vector types
  of given size by doing bunch of scalar operations) was *faster* than the code
  produced with explicit instructions in many cases, or at least comparable in
  its performance.

Furthermore, I've looked into the [source code] of some of the intrinsics (and
of the [`packed_simd`] types). They contain LLVM intrinsics such as `simd_add`.
So, wait a moment… I go to lengths to abstract away the differences between the
instruction sets to hide the fact I have separate intrinsics for each of them.
But then, the compiler just *throws it away* and replaces the separate
intrinsics with a vague „This is addition of such-an-such large vector, figure
out some SIMD in there“. Then it again looks at what instruction set it has
enabled and picks something as it sees best.

Shouldn't there be a more direct way to „You have these instructions and a
vector, figure out some SIMD in there“? And it turns out there is.

The compiler has an optimizing part (is it called a `pass`?) called
auto-vectorizer. This thing tries its best to replace ordinary scalar
instructions with vector ones where it makes sense. And it is pretty good at it.
However, it needs to first prove it *can* replace these 8 scalar additions
with single vector addition.

One can do it by having this `simd_add` intrinsic, that is not exposed to mere
~~mortals~~ crates. But in fact, it is enough if the compiler sees that there'll
be exactly 8 scalar additions (or how many scalar instructions that particular
vector instruction corresponds to) and that the 8 inputs and 8 outputs will be
well aligned in memory. It can figure out some more complex things too, like
shuffles and some masked instructions. It probably can't do *all* the weird
instructions that are available (I still haven't seen it produce a
[`gather`](https://doc.rust-lang.org/core/arch/x86/fn._mm_i32gather_ps.html),
for example, and I haven't even tried this
[drunkenly swaying one](https://doc.rust-lang.org/core/arch/x86_64/fn._mm256_addsub_ps.html)).

## Slipstream cheats

So, instead of going through the very indirect way of intrinsics, I've decided
to write the vector types in a way the auto-vectorizer would like them and then
let the rest for it. In principle, a `u32x16` is „just“ a well-aligned `[u32;
16]` and its addition operator is „just“ a for cycle over the elements of the
two input same-sized arrays. There are some grudging uninteresting details, like
using the [`generic-array`], annotating the operator methods with `#[inline]`
and such, but the idea is the same. This simple thing makes the code much more
tasty to the auto-vectorizer and it does the real magic.

That's why the crate is called [`slipstream`] ‒ similar to the real thing, it
doesn't make anything go faster by applying any force to it, it just removes
some friction and the engine can work better.

## Runtime instruction set detection

You might be asking by now where the idea of using the best instruction set the
CPU knows went.

So if you need to use the [`target_feature`] annotations *anyway* even if you
use the intrinsics, you just get the instruction sets from there in the
auto-vectorized code as well.

And after publishing [the previous post][SIMD library plans], I got contacted by
[Caleb Zulawski]. It turns out he wrote a great crate to solve the awkwardness
of using the [`target_feature`] thing called [`multiversion`]. So [`slipstream`]
doesn't offer any direct support for dealing with instruction sets, but it works
quite well together with [`multiversion`]. The [documentation] and the
[repository] contains some examples of the use.

## What if one still wants to be explicit about the vectorization

If you really wanted to be explicit, the fact I've gave up on that part might be
a disappointment. I'd direct you to the [`safe_simd`] crate-in-development, also
from [Caleb Zulawski]. Originally, we planned to somehow unify our efforts
eventually, but I guess my change of mind about going the auto-vectorized way
put a stop to that. I still think both experiments are worthwhile, so give his
crate a glimpse too.

## How good is it, really

I've tried to write some benchmarks (they are in the [repository]). Both from
the results that are there and from the process of writing them, I reached the
conclusion that „it varies“.

In most cases, I've found some way to write the code around the vectors that
produces result of the same speed as manual vectorization by directly using the
intrinsics, sometimes even faster. So on one side, it is possible to really
improve the performance of the program by using the library.

On the other hand, a lot of versions were slower than the best one and sometimes
slower than the „naïve“ scalar implementation. There were several reasons for
the slowness ‒ sometimes things didn't get inlined properly, „losing“ the
permission to use better vector instructions. Sometimes, on some CPUs (I'm
looking at you, AMD Buldozer) using one AVX2 instruction is *slower* than using
two SSE instructions. Sometimes the code got vectorized, but for whatever reason
the compiler started to juggle between the registers ‒ the code was full of move
instructions, and even though these were vector move instructions, they added
overhead. Sometimes it got weirdly half-vectorized ‒ some operations were
vectorized and some weren't.

I've seen the same code, compiled by the same compiler, be significantly faster
than the naïve one on one CPU and *slower* on another. Smells of arcana a bit.

I don't know how stable the results are across different compiler versions.

Therefore, when using the library, some experimentation and benchmarking is
needed.

However, I have an intuition that some of these problems would still be
happening with manual intrinsic use or with explicit vectorization library. I
also believe some of these problems can be improved ‒ eg. by having better
control over inlining. And maybe someone is able to twist the internals of the
vector types a little to produce better results, or to produce them with better
odds.

## Help wanted

There's some actual „work“ to do on the code itself ‒ it's rigged with `TODO`
comments and such. The API surface definitely is not complete. Some of these are
listed in the crate [documentation].

But these are the things I'll probably get to over time myself if nobody else
does it first (I'll still appreciate the help, though, and with more people it'd
get done sooner). The most valuable thing you can help with is trying it out,
reporting what works, what doesn't ‒ in the form of issues or by submitting more
benchmarks. As said above, I don't really know if this approach is going to be
usable in bigger programs and if it would be practically useful, or if it's only
good enough for summing up a big array of numbers.

In other words, I don't know your use cases, but I'd like to support them if
reasonably possible. Or at least find out that this approach is a dead end.

Actually, what I would really like is for [`packed_simd`] to become usable on
stable Rust so this thing is no longer needed. I'll be happy to deprecate the
thing in favor of [`packed_simd`] or something like that eventually.

[SIMD library plans]: {% post_url 2018-06-28-signal-hook %}
[slipstream]: https://crates.io/crates/slipstream
[`core::arch`]: https://doc.rust-lang.org/nightly/core/arch/index.html
[`target_feature`]: https://doc.rust-lang.org/nightly/core/arch/index.html#dynamic-cpu-feature-detection
[source code]: https://doc.rust-lang.org/nightly/src/core/up/stdarch/crates/core_arch/src/x86/avx.rs.html#46-48
[`packed_simd`]: https://crates.io/crates/packed_simd
[`generic-array`]: https://crates.io/crates/generic-array
[`multiversion`]: https://crates.io/crates/multiversion
[`safe_simd`]: https://github.com/calebzulawski/safe_simd/
[Caleb Zulawski]: https://crates.io/users/calebzulawski
[repository]: https://github.com/vorner/slipstream
[documentation]: https://docs.rs/slipstream
