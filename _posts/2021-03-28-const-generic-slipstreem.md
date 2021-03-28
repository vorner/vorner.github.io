# Using const generics in slipstream

Some time ago, I was [experimenting with SIMD „by
cheating“][slipstream-cheating]. The [library][slipstream] (called `slipstream`)
offers the vector types. These are little fixed sized arrays which correspond to
the registers in the CPU.

Unlike the „real“ libraries ([`packed_simd`]), it doesn't however force the SIMD
by explicitly using the compiler intrinsics. The vector types are really _only_
fixed sized arrays with the right methods and with forced alignment properties.
The compiler has enough information to prove it can vectorize the code and
oftentimes does so in a good enough way.

The advantage is there's much less „magic“ inside the library code and it works
on stable. The disadvantage, the auto-vectorizer doesn't always do the best job
and can even make the code somewhat slower than the original.

The recommended way is to combine the library with something that generates
multiple versions of the functions and picks the right one by runtime CPU
feature selection, like the [`multiversion`] crate.

## The challenge

Since the size of the registers depends on the CPU feature level, it is not
known in advance. It is possible to use larger ones (and the compiler will just
emulate it by using multiple registers and operations). But as there are several
register sizes and several basic types, each combination with its own alignment,
there's a large list of combinations that make sense.

The user is presented with convenient type aliases, like `f32x4` ‒ a vector of 4
32 bit floats. But the library has to somehow offer all these types, even though
they are _mostly_ the same.

The original (0.1) version used the [`generic-array`] library to hold the actual
data and types like these to force the alignment:

```rust
#[repr(align(64))]
Vector64<B, S>
where
    // Some uninteresting and ugly type bounds
{
    data: GenericArray<f32, U4>,
}
```

And then used transmutes, pointer casts and memory copies to operate these.
While it got the job done, the code could only be described as _hairy and ugly_.
Also, there was more `unsafe` around than felt necessary. It was easy enough to
reason about, but it still didn't feel right.

## Welcome const generics

I guess the whole post is about this. The arrival of [const generics] arrival
made it possible to simplify the code *a lot*. In combination with these, the
alignment is not an attribute directly on that type, but is brought in through
another marker type (a zero-sized one). It looks like this.

```rust
#[repr(C)] // So the alignment ZST is at the start
Vector<A, B, const S: usize> {
    // Forces alignment, but as it is ZST, it does not take any space.
    _align: [A; 0],
    data: [B; S],
}
```

The code is much shorter, easier to follow and reason about. Due to that, the
new version brought some more features (nothing big) and the types dereference
to the fixed sized arrays, not slices, which might be more convenient for some
code.

There's still some `unsafe` around, though. It is needed to handle
initialization of the arrays, so it deals with [`MaybeUninit`]. There might be
_nicer_ ways to do it, but the library tries to use as primitive ways as
possible so the auto-vectorizer has an easier way and works more often. This is
one place where using old-style index loops instead of iterators is faster. I
guess this is because the auto-vectorizer is written with C/C++ code in mind (it
lives somewhere in LLVM) and the range checks are trivially removed, since they
are to a fixed-sized constant (passed as the generic parameter).

## Should you use the crate?

Well, that varies. It is mostly an experiment from my side. You can try it, it
should not outright break anything. But always measure the results.

Sometimes, you can get quite good speedups. Sometimes, you get nothing, mostly
because even your original code was already auto-vectorized (that thing is quite
powerful). Sometimes, you may get a slow down, because the auto-vectorizer gets
confused and it starts „juggling“ the registers (I'm not entirely sure what
happens and why, but the resulting code is full of moving values between
registers there and back).

Anyway, the API is quite close to what you get with [`packed_simd`], with the
exception of the [`vectorize`] family of functions. So if you rewrite it with
[`slipstream`] and it is slow, you can try replacing it with that and see if it
helps.

The advantage of [`slipstream`] over [`packed_simd`] is however being able to
compile on stable. You can consider it a best-effort temporary solution before
the explicit packed SIMD support matures.

## Big thanks

Nothing of this would be possible without the work of… a lot of people, I guess.
It's not only the Rust work on const generics that took some time and effort.
There's also the magic in LLVM, years of accumulated smartness and research. And
most of it is quite invisible. It wouldn't be possible without the people that
_use_ it either.

So, thanks ☺, you're great, whoever you are.

[slipstream]: https://crates.io/crates/slipstream
[`slipstream`]: https://crates.io/crates/slipstream
[`packed_simd`]: https://crates.io/crates/packed_simd
[`multiversion`]: https://crates.io/crates/multiversion
[`generic-array`]: https://crates.io/crates/generic-array
[const generics]: https://github.com/rust-lang/rust/blob/stable/RELEASES.md#language
[`MaybeUninit`]: https://doc.rust-lang.org/nightly/std/mem/union.MaybeUninit.html
[`vectorize`]: https://docs.rs/slipstream/0.2.0/slipstream/iterators/trait.Vectorizable.html#method.vectorize
[slipstream-cheating]: {% post_url 2020-06-21-simd-by-cheating %}
