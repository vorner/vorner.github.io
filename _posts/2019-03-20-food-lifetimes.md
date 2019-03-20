# Food model of lifetimes

Sometimes I see confusion about what lifetimes in Rust are and how they act.
This writeup tries to present an *intuitive* model about them. The model might
be inaccurate at places and even a bit ridiculous, but it should help the
subconsciousness to grasp them.

## What lifetimes try to solve

In the past times, when people still had no choice but to use C for all their
coding, they had to manually track when a pointer is no longer usable, because
it points to some portion of memory no longer containing the original variable.
If they made a mistake, bad things like [Undefined Behaviour] happened.

Similarly in the past times (at least where I live), food didn't have expiration
dates written on it. People had to manually track when a bit of cheese or some
apples were no longer good to eat. If they made a mistake, bad things like smell
or food poisoning happened.

## Lifetimes are expiration dates

So we have our parallel. A lifetime specifies when values of the type can still
be used and after which time you better make sure they are not around at all.
References to rotten data are just… urgh.

Notice that lifetimes are properties of *types* not *values*. `&'a str` and `&'b
str` are different types. Also, the lifetime doesn't say how long you'll keep
the *value* around, only how long you *could*. Apples, when stored properly, can
be kept around for about a year (depending on the kind of apple) but you're
going to eat some of them sooner and some later. That doesn't change their
expiration dates.

## Composing things together

This one is simple. If there's a type built from other types, its lifetime is
the shortest one of the parts. If you put apples and cheese and oranges together
to make a meal and put it all into the fridge and forget about it, the first
thing to go bad makes the *whole* thing inedible. And it's the same with types.

## The `'static` lifetime

So, what does it mean for a type to be static? It's a type without any
expiration date. There are foods like that too. You can keep salt around mostly
as long as you want (the fact it *has* an expiration date written on it is just
stupid result of the laws ‒ they say *everything* must have expiration dates,
but it was buried in the ground for millions of years and didn't spoil yet, so
it might be OK for some time yet).

The `'static` types are the same. They don't spoil. It doesn't mean you *have
to* keep them around forever. Like with the salt. But you can eat them sooner
than that. If some API wants to be given some `'static` type, it means the
library doesn't want to bother tracking the expiration dates.

That doesn't mean they need to be constants compiled into the program. It just
means you are not limited on how long you can keep them.

This confusion comes from things like `&'static str`. For the *reference* not to
spoil on its own, the data it *points to* needs to be kept around forever.
Therefore the `str` (not `&str`) needs to be a constant compiled into the
program (or something similar that never goes away). But you still can throw the
reference away sooner and both the `str` and `&str` are types with expiration
date in the infinite future.

## Functions and cookbooks

Functions are odd in regards to references. Functions usually don't have
lifetimes of their own (closures can, though, they can contain lifetimes in
whatever they capture). However, they can accept parameters and return values
with lifetimes. Usually, the output lifetimes somehow depend on the lifetimes of
inputs.

They are in a sense a cookbook. They describe how to create some meal. The
expiration date of the meal might somehow depend on the expiration dates of the
ingredients. If you put old apples in the apples and oranges salad, it's going
to last shorter in the fridge than if made from fresh ones. But the cookbook is
still the same, no matter when or with what apples you use it. The cookbook
itself doesn't spoil.

Closures with captured lifetimes are little bit harder to model using this ‒
it's something like a cookbook written on edible paper that also can spoil, just
like the ingredients.

## Conclusion

Does this help you fight the borrow checker? Probably not directly. It doesn't
tell you how to change your program. But it just might help you make sense of
why the lifetimes were designed the ways they are and what the compiler tries to
tell you.

[Undefined Behaviour]: https://vorner.github.io/undefined.html
