# Mental models around Ok-Wrapping

I'm a bit reluctant writing this, because it's about a controversial and
sensitive topic. Yet, after two days of sleeping on it, I hope this'll hopefully
not cause any more heated discussions and may help some mutual understanding.

This is in part triggered by this post by a blog post by
[withoutboats](https://boats.gitlab.io/blog/post/why-ok-wrapping/), in part
by some twitter exchanges. So, let me start with this, as a response to
withoutboats and everyone in the „Ok-wrapping camp“.

I very much value what You do for Rust. I appreciate all the great work,
some of which I know is done by You and probably much that I enjoy that I don't
know was done by You. Without people who passionately love the language like
You, Rust wouldn't be what it is. Therefore, I thank You. I thank You even for
proposing and supporting the Ok-wrapping ideas, even though I disagree with
them. I know You do so because You care about the language and because You try
to make it better. I thank You and You have my great respect. I usually try to
respond in a professional manner, but if in the past I've failed in that and
some of my answers were more passionate, I apologize. I also consider Rust dear
to me and therefore can react emotionally.

As text doesn't carry tone of voice properly and I'm not a native English
speaker, I'll add this just to make sure: I mean the above literally, not as
sarcasm.

Despite all the respect and gratitude, I still keep my opinion that, *to me*,
Ok-wrapping would make the situation worse, not better. So here I'd like to take
a step back and use some analytical approach of where this disagreement comes
from. I know withoutboats expressed their intent not to continue pushing for the
idea, yet others might want to and I hope both sides could benefit from
understanding the other one ‒ if not to resolve the disagreement, than at least
to gain some perspective and mutual respect. I'm not that optimistic about
this leading to finding a third solution that suits everyone, but one can always
hope.

After reading the above blog post, I think I understand (at least to some level)
the mental model where Ok-wrapping fits well. I still don't *operate* in it, but
I can see why someone who does would see Ok-wrapping would as a good fit or even
a missing piece that should be filled in.

## Fallible and infallible functions

So, this is the world in which Ok-wrapping works. In this world, functions are
either *fallible* or *infallible*. A function that's infallible always
guarantees to provide the asked for answer (let's pretend for today that there
are no such things as panics nor infinite loops). In a mathematical terminology,
the function is called *complete* (I hope; it's some time I've had my last math
class and it was in Czech, not in English).

A fallible function, however, can refuse to answer given some inputs. For some
inputs you get an answer and for some you don't. Again, let's simplify and
pretend functions are pure, or that we consider the „state of the world“ part of
the input parameters so we don't have to dig into details about how this works
with `std::fs::read` and similar. Simply, functions have a hidden `&mut World`
parameter, or something like that.

If you have a function like this:

```rust
fn sqrt(number: f64) -> Result<f64, NegativeInput>
```

then, in this model you say and think that this function returns `f64`, but for
negative numbers it doesn't have an answer for you. Negative numbers simply are
not valid inputs for it and the function is defined only for half of the real
numbers.

And indeed, if you already think that this function returns `f64`, then the
whole stuff around `Result` and `Ok(3.141592)` is just an *annoying
technicality* that's getting in the way of expressing the idea. In this model,
writing something like this:

```rust
fn sqrt(number: f64) -> Result<f64, NegativeInput> {
    42.0
}
```

makes total sense (the point isn't about the particular buggy number; please
ignore that, we'll fix this bug in some future version of the library) and
aligns well with the thinking.

## The operations report model

(I've just made that name up)

Imagine you hire a scientist to evaluate the feasibility and ways to use a
cigarette smoke to produce gold. You certainly *hope* they will give you the
ways to do it, because then you can make a lot of money. And, after some time,
you receive a report stating it is not possible to produce gold from cigarette
smoke. While dissatisfied with the result, you wouldn't say the scientist
*failed*.

As another example, let's have the following function:

```rust
fn check_credentials(username: &str, password: &str)
    -> Result<Privileges, NotAuthorized>
```

If the function returns `Err(NotAuthorized)`, I wouldn't say it failed. I'd say
it's doing its job very well indeed.

Therefore, in the mental model I use, functions don't *fail*. There are no two
separate categories of functions, all functions are *complete* in the
mathematical sense (still ignoring panics and infinite loops). The above
`sqrt` function is defined on the whole axis of real numbers, producing either
`Ok(f64)` or `Err(NegativeInput)`. Both the input set and the output set are
bigger.

If there's any failure, it is not tagged to the function but to the resulting
value. The task of the function is to produce this „operations report“ that may
describe the desired outcome or it might describe why it wasn't possible. This
model has some nice properties which I take the liberty of describing few
paragraphs below.

This might sound like hair splitting, but try to think about it. I actually
believe this is very crucial detail with a lot of consequences of how one thinks
about the code.

My `sqrt` function doesn't return `f64`, but `Result<_, _>`. Not only formally,
but in my head too.

I don't know *why* I prefer this particular model. Maybe it's because I don't
have to make special categories of functions, saving some mental bandwidth to
reason about them. Maybe because I'm of a pessimistic nature (it's said most
programmers are inherent optimists) and the fact that the `Ok` values are more
*prevalent* doesn't make them more *important* or *special*. Yes, the
statistical distribution might have performance consequences. But it's my job to
consider all the error cases too and when thinking about what I can expect from
some subsystem, I usually spend *more* time thinking about what can and will go
wrong ‒ the „happy path“ being mostly an afterthought.

Part of the reason why I like Rust is being able to use this particular mental
model. Languages that have exceptions *force* me into the fallible-function
model, which doesn't come to me naturally (and until being nudged to it more
explicitly by the above blog post, without really understanding what I don't
like about it). I take Rust to be a rare alternative.

Back a little bit. I want to put this into the perspective of how much the
Ok-wrapping *doesn't* work in this mental model and why people using this model
might have a strong reactions to it.

Imagine following code:

```rust
struct Meters(u64);

#[new_type_wrapping]
fn distance() -> Meters {
    128 // Due to the attribute, it gets auto-wrapped into Meters(128)
}
```

If you are like me, your mental type checker just died. Obviously one can't
return 128 and pretend that it's of the `Meters` type.

For me, in this particular mental model, Ok-wrapping is very much the same
thing. Because, in this model, the example `sqrt` function *doesn't* return
`f64`. It returns `Result<f64, _>` and `42.0` *is not* a `Result`. I guess I'd
learn to accept the exception that, for `Result`s, the types don't sometimes
match, but that would be an irregularity in otherwise quite regular and
well-behaving language.

So, for a long time, we had people using the fallible-function mental model,
where it makes total sense and people using this or some other model where it
totally doesn't make sense, each one not understanding how the other side can
still not understand them. Yes, I can see how this is frustrating for both
sides.

Note that the `?` operator fits perfectly well into *both* models. For me, `?`
is a short-circuit flow control operator. Basically a syntax sugar for match &
return, because that pattern appeared in common code very often. Therefore it
deserves a syntax shortcut (I sometimes miss the `¿` operator to short-circuit
in the `Ok` case). For similar reasons I'm not really against having a helpful
macro expanding to something like `return Err(e.into())` (I'd prefer it not to
be called `throw`, as in this mental model exception terminology is mostly a
lie).

## Monads?

I've heard monads mentioned around error handling and futures and streams and
that futures and results are somehow similar.

I have an admission to make. While I was able to write things like GUI
applications in Haskell and generally liked the language, I've not yet seen an
explanation of what a monad is that I could understand. Monads are just too
abstract for me to grasp, let alone reasoning about. This obvious lack on my
side never stopped me from using either Haskell or Rust.

Nevertheless, it prevents me from seeing any kind of similarity between a Result
and a Future. For me, these are really very much dissimilar ‒ Results are inert,
already finished things. These are products, values, objects. Futures are
potentially active, alive. Plans, processes, computations.

So there might be another third mental model around monads, but that one eludes
me completely.

## Some nice properties of the operations report model

So, do I get something else than one mental drawer saved by not having a special
function category (and maybe a keyword not used up)?

Similarly as in functional programming where functions and closures are „just
values“, here results are „just values“. This opens up some new possibilities.

I still haven't figured how to map things like
`cached_results: HashMap<u32, Result<String, Error>>` into the fallible function
model. In this case, there's no function that could be fallible. Therefore, to
do such thing one has to mentally switch the models to something else and
„convert“ the fallible function's result into a „frozen“ representation. This is
perfectly natural in the operations report model, because by the time any
function has finished the result is already frozen.

I can go even further and do something like this:

```rust
fn some_operation() -> Result<Result<u32, E1>, E2>
```

Again, I have no idea how to map this into the fallible model. This might be
because I'm not familiar with it enough. But I believe this nesting is not
expressible in languages that primarily use exceptions as their main error
handling mechanisms and I believe the fallible function mental model and
exceptions are closely related.

Nevertheless, I think *is* useful to be able to express.

One example is:

```rust
fn rpc_call(remote_procedure: ProcedureName)
    -> Result<Result<(), ProcedureFailed>, ConnectionError>
```

Here, I *can* flatten them (maybe using an enum) to one level, but I don't
*want* to, because these errors happen at completely different levels. Being
able to convey this multi-layerity through the API feels useful to the reader.
This is lost in the flat representation, eg. in
`throws (ProcedureFailed | ConnectionError)`.

Another example is described in this blog post about [sled error
handling](http://sled.rs/errors), where unwrapping the errors one by one has
nice effect on overall application robustness.

Needless to say that I would consider this to be confusing to read (no matter
which specific Ok-wrapping proposal it would be). What would it mean in the
fallible model anyway, that the function successfully failed?

```rust
#[try]
fn some_operation() -> Result<Result<u32, u32>, u32> {
    Err(42) // Note: this actually turns into Ok(Err(42)) due to Ok wrapping
}
```

## Conclusion?

Honestly, I don't know if there's any. I just hope to illustrate some reasons
why I personally don't like the Ok-wrapping in as analytical way as possible. I
hope we can at least learn to agree to disagree on this point and respect that
each one has their reasons for the preferences.

I think there might be at least some solutions for the „editing distance“ use
case (changing the return type from `T` to `Result<T, _>`) ‒ maybe `rls` could
offer a function to find all return sites and wrap/unwrap each of them in given
text (this could also cover changing the return type from `u32` to `Meters` from
the above example, or wrapping all return sites in `validate_result_integrity`)?
