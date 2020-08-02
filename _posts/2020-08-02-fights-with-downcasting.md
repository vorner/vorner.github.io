# Fights with downcasting

It is said that Rust beginners fight the compiler. While I might disagree with
the notion of *fight* and would prefer to call it a *dialogue* or maybe a
*dispute*, this story shows the particular activity is not restricted only to
beginners.

This dialogue was not about the lifetimes and borrowing, we (me and `rustc`)
have settled our differences on that one already, but about downcasting. I'm
sharing this partly because it might save someone some time (I did find a
solution, eventually, and you're free to get inspired by it), it might be a good
entertainment for some (some people like to read about others' difficulties) and
maybe someone would also get inspired and figure out a solution to make this
particular thing *easier* by improving the standard library or the compiler in
some way (I don't have a specific proposal, only a pain point).

I can't show the actual full final code (it's not open source), but I'm sharing
snippets demonstrating the point.

## What is downcasting

There are two general approaches to handling values of distinct types uniformly
in Rust. The first, more common one, is through monomorphization, also called
generics or static dispatch. The compiler copy-pastes the code and substitutes
the concrete type for each one. This gives one a lot of flexibility, because the
types everywhere can be substituted and the compiler knows how to handle them ‒
most importantly, it knows how large the values are to store them on stack. It
also knows what exact method will be called, which means they can be inlined,
which might eventually produce slightly faster result. It also produces larger
binary.

The downside is, *everything* all the way up needs to be monomorphized. If you
have a trait `X` and types `A` and `B`, static dispatch won't help if you want
to put both `A`s and `B`s into the same `HashMap` or `Vec`.

The solution comes in the form of dynamic dispatch. That way one doesn't store
the values directly, but stores pointers to the data together with pointers to
vtables ‒ lookup tables that are used to decide what method implementation to
call on that particular value. Then one can have things like
`HashMap<String, Box<dyn X>>`.

While `dyn X` is a type in its own right, it has an unknown size during
compilation. This makes it hard to manipulate ‒ it can't be stored on stack,
only through pointers (boxes, `Arc`s, references, …). Furthermore, for `dyn X`
to actually exist, the `trait X` can't be too rich (this is called [object
safety]). In particular, it disallows using `Self` and associated types as these
would make the caller really confused ‒ all methods (dispatched through the
virtual table) must have the *same* signature no matter what the real type of
the thing is. So this won't work:

```rust
trait X {
    // Every Data could be different
    type Data: Data;
    // How large chunk of stack do I reserve for the result?
    fn create_data(&self) -> Self::Data;
    // What do I pass to this one?
    fn process_data(&self, data: Self::Data) -> Result<(), Error>;
    // What do I pass as `other`?
    fn is_similar(&self, other: &Self) -> bool;
}
```

While the `self` is hidden behind the `Box` or other pointer and can be of
different types (and the compiler with the vtable trick will make sure to cast
it to the specific type for the method behind the scenes), the `other` would
require the *caller* to give it the right type which might be different every
time ‒ which, obviously, doesn't work:

```rust
for v in hash_map.values() {
    // Uh, so, do I pass &A or &B here? And, can someone implement
    // X for some completely different type?
    v.is_similar(/* ??? :-( */);
}
```

Furthermore, object safe traits can't have generic methods either (the compiler
would have to somehow monomorphize all the implementations through the vtable,
even for types it doesn't know about yet).

What one *can* do is this:

```rust
trait DynX {
    fn create_data(&self) -> Box<dyn DynData>;
    fn process_data(&self, data: Box<dyn DynData) -> Result<(), Error>;
    fn is_similar(&self, other: &dyn DynX) -> bool;
}
```

However, there's a slight problem. It's now up to the caller to make sure the
right type is passed in and it is up to the method to deal with casting back to
concrete type and to somehow protect itself from the error where someone passes
something wrong. And this is called downcasting ([on Any], [on Box], [on Arc]),
it's the reverse of type erasure.

So, one would *want to* do something like this:

```rust
trait DynX: Any {
    ...
}

impl DynX for A {
    fn is_similar(&self, other: &dyn DynX) -> bool {
        if let Some(other) = other.downcast_ref::<A>() {
            // Now we can be sure that it is really &A
            self.is_really_similar(other)
        } else {
            false // Different type is definitely not similar
        }
    }
}
```

This, however, doesn't work. I'll explain why below. But first a small
sidetrack.

## Alternative approach with enums

The `dyn DynX` approach is *open* in the form one doesn't need know what types
it'll work with in advance. If one knows that there are going to be only few
concrete types, it is possible to deal with the problem with enums, something
like this (we'll assume there will be only two types):

```rust
enum Either<A, B> {
    A(A),
    B(B),
}

impl<A: Data, B: Data> Data for Either<A, B> {
    ...
}

impl<A: X, B: X> X for Either<A, B> {
    type Data = Either<A::Data, B::Data>;
    ...
    fn is_similar(&self, other: &Self) -> bool {
        match (self, other) {
            (Either::A(s), Either::A(o)) => s.is_similar(o),
            (Either::B(s), Either::B(o)) => s.is_similar(o),
            _ => false,
    }
}
```

In many cases this is easier. The problem is when there are *a lot* of types
that implement the trait or if they are going to be added often or even added by
third parties (plugged in from outside). The enum approach is *closed*, adding
more means modifying the enum (or doing nasty things like trees from the above
`Either` type). It however uses the better supported static dispatch.

## The problem I was solving

I had some traits that were not object safe. I had several types implementing
them already and didn't want to modify the traits with the above trick with
passing `dyn Something` around, mostly to make all the current and future
implementations simpler.

But I wanted to put them into containers and keep them around. So I've decided
to write another set of similar but object-safe traits and implement wrapper
types for them. Something like this:

```rust
trait DynData: Any { }

impl<D: Data + Any> DynData for D { }

trait DynFactory {
    fn create_data(&self) -> Arc<dyn DynData>;
    fn process_data(&self, data: Arc<dyn DynData>) -> Result<(), Error>;
}

struct DynWrapper<F> {
    factory: F,
    some_other_fields: WhatEver,
}

impl<F: Factory> DynFactory for DynWrapper<F> {
    fn create_data(&self) -> Arc<dyn DynData> {
        // Calls the statically-dispatched Factory::create_data and lets rust
        // coerce Arc<F::Data> into Arc<dyn DynData>
        self.0.create_data()
    }
    fn process_data(&self, data: &dyn DynData) -> Result<(), Error> {
        // It's part of the contract we get *our* data
        let specific = data
            .downcast_ref::<F::Data>()
            .expect("Not my data!");
        self.0.process_data(specific)
    }
}

// A lot more boilerplate...
```

## This doesn't work

If you paid a *very* close attention, you've noticed that all the downcasting
methods are specifically for `dyn Any` or `Box<dyn Any>`. The `Any` trait itself
has only [`type_id`]. But we don't have `dyn Any`. We have `dyn DynData`,
therefore the `data.downcast_ref()` won't compile. So, what can we do?

## The big hammer ‒ use unsafe

When one is willing to use these heavy hammers, it is possible to just implement
`downcast_ref` ourselves. We *do* have the [`type_id`], so we can check if the
type is what we expect it to be. Then we can just cast the pointers around a
bit, read the [Rustonomicon], read the [Unsafe guidelines], remove
`#![forbid(unsafe)]` from our project's header and prove that we didn't make any
accidental mistakes (like extending a lifetime past its expiration date or
creating multiple mutable references). After all, it's what `downcast_ref` [does
internally](https://doc.rust-lang.org/src/core/any.rs.html#195-204).

But let's assume we are not really comfortable with such approach.

We could use [mopa](https://crates.io/crates/mopa) instead, as it simply wraps
the above in a crate, but it hasn't been updated for 4 years and seems
unmaintained. I didn't feel comfortable with that even though the code there is
probably correct. Oh, well, let's continue our search.

## Let's keep `Arc<dyn Any>` around

So, instead of keeping `Arc<dyn DynData>`, we could keep `Arc<dyn Any>`, right?
Well, only if the `DynData` didn't have any methods we would still want to call.
If we wanted to call some methods from `DynData` (the trait in the example is
empty, but pretend I've put some method in there too), we would be out of luck.
Once we have `dyn Any`, we can get a specific type of the data out (let's say
`AData` or `BData`), but to do that, we would have to list all the specific
types and try one by one. No good. Downcasting to `dyn DynData` won't work,
because while `&AData` can coerce into `&DynData` because it implements the
relevant trait, it *isn't* `DynData`.

```rust
if let Some(adata) = data.downcast_ref::<AData>() {
    adata.data_method();
} else if let Some(bdata) = data.downcast_ref::<BData>() {
    bdata.data_method();
} else if ...
    // This doesn't seem to scale particularly well...
```

```rust
// This would panic at runtime. Actually, it would *if it compiled at all*.
let data = data.downcast_ref::<DynData>().unwrap();
```

If we really wanted to go this way, we could keep *both* `Arc<dyn Any>` *and*
`Arc<dyn DynData>` around. While this is possible to create, it is cumbersome,
carrying both around.

```rust
let data = Arc::new(self.0.create_data());
let any_data = Arc::clone(&data) as Arc<dyn Any>;
(any_data, data as Arc<dyn DynData>)
```

## Cast to supertrait first!

Ok, we don't want to *keep* two copies of the `Arc` around as two different
trait objects. But what about converting one to the other? We've already decided
that we can't make `dyn DynData` out of `dyn Any`, but what about the other way
around? After all, the traits inherit, so we should be able to *upcast*, and
indeed this code seems to compile:

```rust
let any_data = &data as &Any;
let data: &F::Data = data.downcast_ref().expect("Not my data");
```

However, this panics at runtime. The reason is, instead of creating an `Any`
that has a `AData` inside (the very original type), it creates an `Any` that has
`Arc<dyn DynData>` inside. Other ways of casting, like `data.deref() as &dyn
Any` produce a compilation error. So instead of of trait object wrapping the
original data we get a trait object wrapping trait object wrapping the original
data. Doh! That's certainly not what we wanted and it won't help us in achieving
our goal.

## `as_any`

There's, however, one thing that still knows the right type of the data and can
therefore create the *right* `Any` for us and that's the original data itself.
That data still lives *behind* the veil of the trait object's vtable.  We just
need to ask that one instead.

The trick how to do that is to inject a new method into the `DynData` trait, one
that is best called `as_any`:

```rust
trait DynData {
    ... // Other methods
    fn as_any(&self) -> &dyn Any;
}

impl<D: Data + Sized> DynData for D {
    ...
    fn as_any(&self) -> &dyn Any {
        // The right coercion happens here, because Self is the *right* type
        self
    }
}
```

And then we can do the downcasting to our pleasure:

```rust
data.as_any().downcast::<F::Data>().expect("Not my data");
```

## Naming

The object safe traits were named `Dyn*` here. It's not to suggest that they
*should* be named that way, and I *didn't* name them that way in the real code.
However, it felt clearer in context of these code snippets distinguished from
the original non-object-safe traits.

## Takeaways

It somehow feels like dynamic dispatch is a bit of a second-class citizen in
Rust. It is usable, with enough beating of the code, it can be made to do the
right thing.

If you have a better approach than the `as_any` hack, I'd be happy to hear it
and simplify the code. If you have the idea how to make these type dances easier
and want to write an RFC, you have my respect and mental support (though I don't
know if I would find the time to provide some tangible help with it).

[on Any]: https://doc.rust-lang.org/std/any/trait.Any.html#method.downcast_ref
[on Box]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.downcast
[on Arc]: https://doc.rust-lang.org/std/sync/struct.Arc.html#method.downcast
[object safe]: https://doc.rust-lang.org/stable/book/ch17-02-trait-objects.html#object-safety-is-required-for-trait-objects
[`type_id`]: https://doc.rust-lang.org/std/any/trait.Any.html#tymethod.type_id
[Rustonomicon]: https://doc.rust-lang.org/nomicon/index.html
[Unsafe guidelines]: https://rust-lang.github.io/unsafe-code-guidelines/
