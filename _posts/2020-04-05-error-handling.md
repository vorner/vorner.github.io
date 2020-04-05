# My private take on error handling in Rust

I've had a note in my to-do list to write down some of my own thoughts about
error handling in Rust for quite some time and mostly got used to it sitting in
there. Nevertheless,
a [twitter discussion](https://twitter.com/yaahc_/status/1246126703820210177)
brought it back to my attention since I wanted to explain them and honestly,
twitter is just not the right medium for explaining design decisions, with its
incredible limited space and impossible-to-follow threading model.

Anyway, this is a bit of a brain dump that's not very sorted. It contains both
how I do error handling in Rust, why I do it that way and what I could wish for.
Nevertheless, my general view on the error handling is it is mostly fine ‒ it
would use some polishing, but hey, what wouldn't.

And of course, the way *I* do error handling doesn't necessarily mean it's the
way *you* need to be doing it too, this is very much based on personal
preferences as much as some technical reasons. You're free to think I'm doing it
all wrong :-).

## Language syntax

I know this is somewhat contentious topic. But I'm a strong opponent of adding
more specialized syntax for error handling specifically. Currently, error
handling is done through the `Result` type. It's just a type, has some methods,
implements some traits and it *composes well*. You can have
`Vec<Result<(), Error>>` or even monsters like:

```rust
HashMap<String, Box<dyn FnMut() -> Box<dyn Future<Item = Result<Option<u32>, Error>>>>>
```

(that would be a registry of asynchronous handlers of commands, each promising
to eventually maybe return an `u32`, but being able to fail; and I probably put
too few or too many `>`s there, sorry if you get a headache from an unclosed
delimiter)

Any new syntax like `fn x() -> u32 throws Error` makes the connection between
this being really a `Result` (with useful methods on it and being able to be
stored in a `Vec`) longer to grasp without an obvious (to me) advantage.
Furthermore, it promotes error handling into some special place in the language
‒ you no longer could write your *own* fully-featured `Result`, making `std`
more privileged. And it opens the door further to „Should Option also have a
special syntax sugar, so you could write `fn x() -> maybe u32` and should it
compose to `fn x() -> maybe u32 throws Error`? What about `fn x() -> maybe u32
maybe throws Error`? Should we have `locked String` instead of `Mutex<String>`?“

That's my two cents on this, but I really don't want to dive into it more.

So, if anything would be to be added to the language to help with error
handling, I believe it should be of general use and in line with expressing a
lot with types instead of special keywords.

Some time ago I've seen an idea (I believe by Withoutboats, but I might be
mistaken) that error handling would really get better if Rust handled dynamic
dispatch & downcasting in some nicer way. I kind of agree on that front. Let's
see below.

## Open vs. closed error types

We have these leaf error types that describe one specific error:

```rust
/// We failed to synchronize stuff with the backend.
#[derive(Copy, Clone, Debug)]
struct SynchronizationError;

// Some more boilerplate here...
```

That's nice, but what if our function can fail in multiple different ways? There
are two general approaches to that.

The closed error type is if we know all the ways it can fail. Let's say
something in lines of:

```rust
#[derive(Clone, Debug)]
#[non_exhaustive] // Make sure we can add more variants in future API versions
enum PossibleError {
    SyncError(SynchronizationError),
    OutOfCheeseError(MouseError),
    ...
}

// Some more boilerplate here...
```

Well, one could hope for somewhat less boilerplate (that I've excluded here) ‒
and there are crates for that. One could also hope for some way to just list the
damn errors in-line instead of having to create the whole enum out of band
manually, but that comes with a full new can of worms (like creating unnameable
types which make it harder to work with on the caller side) and this isn't
really that bad anyway. And working with these errors is quite nice, Rust really
likes enums:

```rust
match fallible_function() {
    Ok(result) => println!("Cool, we have a {}", result),
    Err(PossibleError::SyncError(e)) => error!("{}", e),
    Err(OutOfCheeseError(mouse) => error("{}: Squeek!", mouse),
    _ => error!("Unknown error"),
}
```

But let's say we don't really know all the ways a function can fail, either
because we are lazy slackers that can't be bothered to track it down and we
don't really care (speed of development *is* a valid reason), or because
somewhere in there there's a user-provided callback that can also fail for
whatever reason our caller likes, so we can't really limit them to our own
preset of error types. That's the open case.

So let's have something like `Box<dyn Error + Send + Sync>` (some people prefer
to wrap that up into another type, but the high-level idea is the same). If we
want to *just* log the error and terminate (either the application, or one
request, or whatever), it's fine. This thing can be printed, because it
implements `Display`. All well-behaved errors do.

But what if we want to check if it happens to be one of the specific error types
we can somehow handle? If our cache fails to load, that sucks, but we can
recover and regenerate it. Now we do something like:

```rust
if let Some(e) = error.downcast_ref::<CacheError>() {
    ...
} else if let Some(e) = error.downcast_ref::<OtherError>() {
    // This is getting tedious
}
// And this doesn't really work, does it?
// else if let Some(e) = error.downcast_ref::<Some|More|Errors>() {
```

Note that this is not a problem of *just* error handling. Any time we get a
`dyn Something`, it's kind of painful. I mean, one should generally not downcast
things in perfect world, but one of the valid reasons to use Rust is because the
situation is not perfect and one *has to* do things that generally should *not*
be done. So, why make it painful? With a very tentative syntax, this would make
it much nicer:

```rust
match e {
    e@dyn CacheError => { ... }
    e@dyn OtherError => { ... }
    e@(dyn Some | dyn More | dyn Errors) => { ... }
}
```

And yes, this syntax probably can't be used because it collides with something
that's valid today and means something entirely different. I want to
demonstrate the idea, not the exact syntax.

## Some history: the failure and spirit crates

Finally moving from the syntax part (which I believe is OK) to the library part.
Let's do a bit of historical context.

I'm the author of the [spirit](https://crates.io/crates/spirit) family of… let's
call it configuration manager helpers. It takes care of loading and reloading
configuration and setting up parts of an application. In that area lives *a lot*
of error handling.

So where does that stand?

* It's a library. It provides some of its own leaf errors.
* Configuration is very much about user-provided callbacks. So we are in the
  open-errors area.
* A lot of these errors are going to be shown to the end user, so they have to
  be nice and *meaningfull*. That means having enough of context for the user to
  figure out what went wrong.

At that time, the [failure](https://crates.io/crates/failure) appeared and it
was the perfect tool for the job, because:

* It has its `failure-derive` sub-crate (enabled as a feature). This cuts down
  on the boilerplate of leaf errors. Just throw in few derive and annotation
  attributes and you're done (I believe the procedural macros & derives is one
  of the big selling points of Rust, it saves *so much* work).
* It has a `failure::Error` catch-all type that handles the open use case really
  nicely.
* Everything gets these nice `.context` calls that *wrap* the error in another
  layer. Eventually, when the error bubbles all the way up, I have a multi-layer
  error and can print something like
  `Configuration reload failed: Couldn't load the Foo Descripotor 'xyz.desc': No such file or directory`.
  That's what the user needs to see.
  (note that, unlike what `failure` proposes in the documentation, I prefer to
  output *all* the levels, not just the top one).

All in all, I believe `failure` was a great success in the sense it showed a way
forward. Nevertheless, it has bunch of drawbacks. Specifically:

* It uses its own replacement of the `Error` trait (`Fail`). There were very
  good reasons for that, but they turned out to be solvable. So today it just…
  doesn't play well with other things.
* Its `Error` type is a new opaque type (that doesn't implement `Error` trait).
  If my library uses that and exposes it through a public API, I force you to
  use it too.
* The `Error` type hard-depends on backtraces. Backtraces for errors are nice.
  However, sometimes they are very much a luxury. I'm not speaking about
  performance. But some of the code I was writing at the time targeted „bigger
  embedded“ ‒ somewhat limited system with a different architecture, but with a
  full `std` support, OS and stuff ‒ imagine a Raspberry Pi style device. This
  means cross compiling. And of all the crates out there, the `backtrace`
  crate is probably the *very worst* to make work when cross-compiling.

## Evolution

After `failure` got more popular than expected and discovering that the reasons
why it didn't use the `std`'s `Error` trait could be fixed, people started to
discuss the ways forward ‒ including `std`-compatible `failure-0.2`, extending
the trait in `std`, etc. And when the Rust community starts discussing
something, it is a very thorough discussion. Which is good because the result is
eventually great. But it *takes ages*. When I need or want something, I need it
*right now*, not eventually. And I needed to move forward with my error handling
‒ I wanted to stop using `failure` for `spirit`.

But I didn't want to tie in into one specific library again, both because
everything was (is) in a flux and the landscape can change and because I no
longer wanted to force anything specific onto my users.

Fortunately, someone did the work and extracted the derive part of `failure` and
modified it to work with the `Error` trait ‒ and
[err-derive](https://crates.io/crates/err-derive) was born. It saves the
boilerplate for leaf error types and closed-enum error case:

```rust
#[derive(Copy, Clone, Debug, Error)]
#[error(display = "Failed to synchronize something with backend")]
struct SynchronizationError;

// No more boilerplate!!
```

(I've been pointed to [thiserror](https://crates.io/crates/thiserror), which
seems to be another implementation of the same thing)

But there was the other half of failure, so I went and wrote a very minimal
[err-context](https://crates.io/crates/err-context) crate.

It provides:

* An `AnyError` type alias. This is a type alias to
  `Box<dyn Error + Send + Sync>`, not a new opaque structure. This is *on
  purpose*. Type aliases are just aliases and they are structurally-equal. That
  means I don't expose the fact that `spirit` is using `err-context` in the
  public API, I expose just this type alias which can come from whatever crate.
  If I decide something better appears, I can switch the thing without changing
  the API. And my users can use whatever other error-handling library, because
  this type alias is based just on `std`. Therefore, it is future-proof and
  plays well with others (the opposite of vendor lock-in).
* Bunch of extension traits and some plumbing types, so the `.context` and
  similar things work (both on concrete types implementing the `Error` type and
  on `AnyError`). So my errors still can have all these layers.
  Nevertheless, you as the user that got the error can iterate through the
  layers without using `err-context`, because it is based on the
  [`source`](https://doc.rust-lang.org/std/error/trait.Error.html#method.source)
  method of the `Error` type. And if you compose your error layers in some other
  way, the helpers to print them to logs or format them from `err-context`
  (re-exported from `spirit`) will work on them too.

However, I don't really try to publicize the `err-context` too much, or to
develop it too much. It works, I use it, but I wait for something more
„official“ to appear eventually. Then I can just deprecate it, because the whole
design is prepared for it getting replaced.

## Why don't I use XYZ?

Sometimes, I get asked why I wrote my own `err-context` instead of using
something else. I believe generally one of these three reasons apply to whatever
XYZ:

* That XYZ is younger. I've already mentioned I'm impatient so I did a very
  minimal thing quite early. I don't see any error-handling crate to be an
  obvious winner *yet*, so I'm still waiting before switching.
* It is quite heavy-weight. Usually, a lot of these crates bring everything and
  kitchen sink. I like the split between the parts, one for the leaf types and
  another for gluing things together. Not having mandatory backtraces is a big
  win for me.
* It uses an opaque type wrapping that `dyn Error`. There might be quite good
  reasons for that (eg. `Box<dyn Error>` doesn't implement `Error` itself, it is
  two words large, etc, etc). But it also leaks through my public API and after
  `failure`, I don't want that again, not unless there's a clear winner in the
  landscape (or even better, such opaque type gets into `std`).

## Things I miss

I've mentioned some of the dynamic matching above as a nice to have. I have some
more things that I'd consider nice from the crates ecosystem:

* If you're a crate author, make sure all your errors implement the `Error`
  trait. It is frustrating when an obvious error can't be casted into any of
  these catch-all-errors types by `?` and one has to do manual conversion
  gymnastics. If you worry about the boilerplate, use `err-derive` or similar.
  It brings all that `syn` and `quote` compile-time dependencies, but almost any
  bigger end-user application has *some* procedural macros anyway, so their cost
  is already paid for.
* The `Error` trait is `std`, but I believe it could go at least to the `alloc`
  level if not directly to `core`. Sometimes I discover crates that are no-std
  ready, but the error types can't follow my previous point because of that.
* If your public API operates with `Box<dyn Error>`, make absolutely sure that
  it's not missing the `Send + Sync` parts. Unless you do something really
  weird, the leaf error types are very nice value types implementing `Send +
  Sync`. But by wrapping it into `Box<dyn Error>` (without the markers), you're
  hiding it and severely limit the way these errors can be manipulated. You
  furthermore force the limitation upwards, making your users use only `Box<dyn
  Error>` making the problem bigger. A lot of the catch-all error types also
  mandate `Send + Sync` and can be created from `Box<dyn Error + Send + Sync>`,
  but not from one without the markers.
* The syntax of `.context()` on results and errors is Ok, but sometimes one has
  to define a closure, call it directly and apply it on that just to attach the
  context. That feels a bit awkward. Some macro or attribute macro for that
  would be nice (I think I've seen something that was *almost* there).

## Conclusion

Overall, I'm mostly happy with where the error handling is already. Some
improvements are *possible* and may make it nicer to use, but I guess it's
mostly a matter of time for one library to take the lead and win, then some time
of polishing. There's nothing I'd be entirely outright *missing* or that some
form of error handling would be *impossible*.

Also, I don't plan to be pulled into endless discussions about error handling.
This is more of a report of what I prefer, not an attempt to start a flame war.
