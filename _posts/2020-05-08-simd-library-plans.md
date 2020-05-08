# SIMD library plans

I believe Rust is a great language to make SIMD actually usable for ordinary
humans. I've played with libraries to making it accessible two years ago (or was
it 3?) and my impression was „Whoa! This is cool. I can't wait until this is
usable on stable.“ The libraries back then were [`stdsimd`] and [`faster`].

Fast forward to today. I considered using some SIMD operations in a project in
work. I have some bitsets and wanted to do operations like bitwise AND on them.
If I represent them as bunch of unsigned integers, using SIMD on that makes
sense. But for that, I need to compile on stable, I want the code to be readable
and I don't want to deal with writing multiple versions of the code to support
multiple levels of SIMD support.

The thing is, while using SIMD on stable is possible, the standard library
offers only the intrinsics. These are good enough as the low-level stuff to
build a library on top, but none of the current ones quite cut it.

* [`packed_simd`] takes the interface of the previous `stdsimd` and provides
  types like `u8x16`. But it still requires nightly, because it depends on many
  unfinished features (const generics, probably specialization) and these don't
  seem to be ready any time soon.
* [`faster`] seems to be dead and not updated. I think it is waiting for what
  comes out of `packed_simd`.
* [`simdeez`] offers only signed ints and is unsound. The interface is a bit
  awkward. I've eventually tried using this one and it is possible, but I
  believe it's possible to do better.

So I think it's time I roll up my sleeves and give it a shot. So here are my
plans.

## What is SIMD and why it's great

CPUs are complex beasts and they try to run as fast as possible. For that, they
do a lot of magic ‒ pipelining, speculative execution, memory prefetching, etc.
So, when there's an instruction to sum up two numbers together, a lot of effort
goes into decoding the instruction, scheduling it onto some execution units,
getting the operands, mapping between real and virtual registers, and only a
little bit into the actual summing. Some clever people decided it makes sense to
introduce instructions that take tuples of operands, do the decoding and all the
other overhead once, but do multiple (2, 4, ...) summing operations in parallel,
because it requires only having wider (or more) arithmetic units. So, instead of
doing this:

```rust
let c1 = a1 + b1;
let c2 = a2 + b2;
```

The CPU can do something like this, but taking the time of only one instruction
(that's a bit simplifying the things, it's never that easy with performance).

```rust
let (c1, c2) = (a1, a2) + (b1, b2);
```

Under optimal circumstances, this allows great speed-ups ‒ depending on how
modern the CPU is and how many „parallel“ operations it can do at once.

## Why SIMD is not so great

The problem with SIMD is, it's hard to use, for several reasons.

* There are newer processors and older processors. The older ones support only
  the older vector instructions, like SSE while newer ones add more modern
  instructions like AVX. The more modern support more parallelism so we really
  want to use them, but we also really want to support the older CPUs, so when
  using SIMD, we end up writing the algorithm multiple times and auto-detecting
  at runtime which one to use. That's annoying. If we want to aim for other
  things than just x86-based CPUs, but ARMs too, we end up writing even more
  variants.
* It's usually available only as compiler intrinsics, which are more or less
  just the low level CPU instructions wrapped into syntax of function calls. So
  working with these is awkward. If we have two arrays of floats, we want to
  write simple iterator-based code with `.zip` or `.sum`, not worry about
  intrinsics. We want the code to be readable, not a submission to the
  [obfuscation contest].
* It's possible to use the vector instructions only when the memory is properly
  aligned, the lanes are independent..

Therefore, SIMD is a lot of work to use. It usually is not worth the effort and
it ends up being used only in very core parts of high-performance code, like
video decoders. Compilers try to use SIMD when possible too, but it's not
perfect because they usually aim at the „common denominator“ instruction set
(what even the oldest CPUs support) and they often fail to prove some of the
properties (like that the size of the slice is a multiple of the number of lanes
or that it's well aligned).

## Two approaches to SIMD in Rust libraries so far

It's a pity only very few pieces of code actually take the advantage of the full
power of the CPUs. If Rust made SIMD usage approachable by people instead of the
current dark arts of few selected, it could lead to making ordinary algorithms
and programs faster. If Rust programs were consistently 5% faster than C++
programs because it's possible to just use SIMD without studying 4 years for it,
it could drive more adoption.

So there appeared two attempts to have an interface which is actually usable in
normal code.

### The Vector Types way

This is what [`packed_simd`] does. It offers bunch of types named like `f32x16`.
These are almost like `[f32; 16]` but with the proper alignment. The programmer
can express the intention of having the slices well aligned and multiples of the
lanes, or maybe have pairs (`f32x2`) when doing 2D graphics. The compiler then
doesn't have to prove anything, the prerequisites are already guaranteed by the
type.

Under the hood, an operation of `f32x16` can translate to one `512`-bit wide
intrinsic, or two `256`-bit wide, or just `16` ordinary instructions, depending
on what the target architecture supports.

The downside is, this doesn't really work well with runtime detection. Doing the
detection inside each `+` between vectors will be slow, we'll likely lose more
than we gain by using the wider instructions. So this compiles into the common
denominator too. This is still quite good ‒ both because this can be overridden
when compiling and some older CPUs ignored, and because for example `x86-64`
always supports at least SSE (`128` bit wide vectors).

### The native vector type

This is what [`faster`] and [`simdeez`] do. While above we had a set of widths
of each base type (eg. `f32x2`, `f32x4`, `f32x8`, `f32x16`) and we picked the
one that made sense for our code, which was possibly wider than what the CPU
does, here we get just the one sized exactly for the CPU. So we might get either
`f32x4` or `f32x16`, depending for which instruction set we compile for
currently.

In this model, it's the job of the programmer to write the code in
width-independent way. If you have a long vector of floats, it gets divided into
pairs or quadruples depending on what is being compiled for.

As the whole closure or function is being compiled for a certain architecture,
it allows the compiler to optimize a little bit more. Furthermore, [`simdeez`]
compiles for multiple instruction sets at once (using macros) and can dispatch
the right variant ‒ it detects the CPU support and then chooses the appropriate
variant.

## My plan

I'd like to combine these two approaches. First, I'd provide marker „Instruction
set“ types, like `Sse4_1` or `Avx`. Then there would be a type alias
`CompileTime` which would be the newest instruction set detected at compile time
(eg. the common denominator).

Then there would be bunch of types like with [`packed_simd`], but parametrized
by the instruction set. So `f32x16<Avx2>` would sum by doing one AVX2
instruction, `f32x16<Sse>` would do it by 4 SSE instructions and
`f32x16<Polyfill>` would do it by 16 „ordinary“ instructions. All of them would
still be more or less `[f32; 16]` under the hood, just with the right alignment
and they would be convertible to each other, just the operations would be done
differently. The bare-bones `f32x16` would default to be `f32x16<CompileTime>`.

The instruction set markers would have bunch of associated types and constants ‒
the „native“ `f32` type, native number of bits, etc. An instance of the marker
type would work as a detection token, see below.

### Safety of the operations

What happens if I do this on an old CPU?

```rust
fn sum(a: f32x16<Avx2>, b: f32x16<Avx2>) -> f32x16<Avx2> {
    a + b
}
```

This would explode in nice fireworks of [undefined behaviour]. So this must not
be possible without use of `unsafe`. But I don't like libraries that force the
user to use `unsafe` ‒ sure, some `unsafe` might be available for special needs,
but ordinary usage should be possible with safe only.

The idea is to gate the *creation* of these vectors on having detected the CPU
support. Doing detection every time one is created would be slow, but it is
possible to return a token when successfully detecting the support and require
the token as a parameter, something like this:

```rust
// Returns Result<Avx2, InstructionSetNotAvailable>
let token = Avx2::detect()?;
let a = f32x16::new([0.0; 16], token);
let b = f32x16::new([0.0; 16], token);
```

### Dispatch of the best variant

So we can detect *if* some instruction set is available and create the
corresponding vectors. But what if we want to let the compiler generate code for
all possible levels and pick the best one at runtime? The answer is generics and
a auto-generated function called into existence by a proc macro.

```rust
#[simd_dispatch]
fn sum<I: InstructionSet>(token: I, a: f32x16<I>, b: f32x16<I>) -> f32x16<I> {
    a + b
}

// Creates vectors of the CompileTime instruction set
let a = f32x16::new([0.0; 16], COMPILE_TIME_TOKEN);
let b = f32x16::new([0.0; 16], COMPILE_TIME_TOKEN);
// Detects the CPU instruction set, generates the appropriate token and casts
// the parameters appropriately.
sum_dispatch(a, b)
```

### Iterators

Sometimes we don't have our data partitioned into vectors. Instead we have a big
slice of the primitive types. The library would provide iterator adapters that
would „cut“ the array into appropriate vectors and return these (or references
to them). There would be multiple variants, eg:

* `iter_vectors_unsafe`: It would split the slice into the vectors, assuming it
  is well aligned and divisible by the number of lanes. This one would be unsafe
  to call and produce undefined behaviour if the assumptions wouldn't hold.
* `iter_vectors_aligned`: Like above, but panicking at runtime if the alignment
  and divisibility is not satisfied.
* `iter_vectors`: Might produce an incomplete vector on the start and end to
  compensate for alignment, filling it with zeroes.

Internally, these would be just type-casting the parts of the slice into the
vectors (except the incomplete vectors above) for speed.

There would be `mut` variants too.

### Behind the scenes

I want this to work on stable. And there'll probably be a lot of very similar
code. So the intention is to generate it either using macros or the `build.rs`
script. The latter might actually be more convenient, using the [`quote`] crate.

## Executing the plan

This seems to be a bit ambitious thing. I don't really want to produce yet
another unfinished library. So, to have some chance of success, I plan to do it
this way:

* I'll try scavenging code from other libraries where possible. I probably don't
  want to directly fork one of them, I'm aiming more at mix-and matching the
  pieces. I'll, of course, give credit where appropriate.
* First I'll try writing some of the code directly, and make the generation work
  after I have an idea how to interlock traits in the right way to make
  everything work.
* I'll try to kick it off with one or maybe two instruction sets with smaller
  number of vector types than the full set. Then I'll ask others for some help
  once the basic skeleton of the library crystallizes a bit.
* Only if this works, I'd pick a name and publish it on `crates.io`, to not put
  something up before making sure this can work.

So, if you want to help, or want to use it, wait a little bit ‒ stepping on each
other wouldn't be beneficial at the start, but I'll definitely let it be known
when multiple people could fit and cooperate on the code.

If you have an abandoned SIMD library and think reusing the name would make
sense, I'm open to suggestions.

[`stdsimd`]: https://crates.io/crates/stdsimd
[`packed_simd`]: https://crates.io/crates/packed_simd
[`simdeez`]: https://crates.io/crates/simdeez
[`faster`]: https://crates.io/crates/faster
[obfuscation contest]: https://en.wikipedia.org/wiki/International_Obfuscated_C_Code_Contest
[undefined behaviour]: /undefined.html
