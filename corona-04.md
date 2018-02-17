---
date: 2018-02-11
category: rust
---
# Corona: If you want to get async out out of your way

For some time now I develop a Rust library for asynchronous programming with
coroutines, called [Corona](https://crates.io/crates/corona) (note there's a
version 0.4.0-pre.1, but Crates prefer the „stable“ 0.3.1). I believe it is
starting to be useful, so I wrote this description to show what it is good for
and how it fits into the big picture of Rust.

There'll be some more changes, though, at least because Tokio just released a
new version (and Futures plan to do so soon), so Corona will have to adapt.

## Motivation and goals

Corona is here to help with asynchronous programming. Asynchronous programming
(eg. handling many tasks at once, most of them waiting for something external to
happen) is already possible with Rust ‒ the biggest player in there is probably
[Tokio](https://tokio.rs).

The problem is, asynchronous programming is *hard*. You have to consider
interactions between all the tasks that may be happening at the same time. You
need to cope with something that you wait for *not* happening (eg. the request
getting lost). The [Futures](https://crates.io/crates/futures) are quite a good
abstraction, but the code written with them often feels convoluted and
unnatural, making the asynchronous programming even harder (it's smaller mental
barrier if you've already heard the name Monad).

This is where Corona comes in. It provides stack-full coroutines that can switch
to other coroutines when waiting on something, making the code feel much more
linear and natural.

What Corona tries to do:

* Make it as comfortable as possible to write correct asynchronous code.
* Integrate with the rest of the ecosystem. It is possible to use all the Tokio
  and Futures related libraries with Corona without much fuss. It is possible to
  mix coroutine code with purely futures-based one on the same thread.
* Allow switching between the paradigms as needed. A coroutine is also a
  future that resolves once the coroutine terminates. And a coroutine can wait
  on a future to resolve.
* Integrate with code that isn't async-aware. It is possible to suspend the
  coroutine from deep within the stack, like a callback, even from within
  recursion. There are wrappers around `AsyncRead/AsyncWrite` streams (eg.
  around a `TcpStream`) that can suspend while waiting for data even while
  passed to functions that don't understand async ‒ like
  [Serde](https://serde.rs) decoding routines.
* Correctly interact with borrowed data. Futures usually need to own all their
  data, polluting the code with `Arc`s and `.clone`s. It is possible to use
  futures that only borrow the data from coroutine's stack with Corona.

What Corona does *not* try to do:

* Attain the best possible performance. While performance is a good thing, it is
  a *secondary* goal for Corona. The execution speed isn't exactly bad (as
  seen in [this benchmark](async-bench.md) and there are patterns where it
  can be even faster than other solutions (parsing large data structures with
  serde ‒ [with many
  approaches](https://docs.rs/tokio-io/0.1.5/tokio_io/codec/trait.Decoder.html#tymethod.decode)
  the data need to be accumulated and retried every time some more arrive until
  the parser says the whole message is received, while Corona can just suspend
  the parser), the cost of using Corona is *memory consumption*.
* Directly port paradigms from other languages (eg. Go) to Rust. There might be
  similarities, but feeling *native* to Rust is more important than just copying
  what others do. This touches to the topic of threads a bit. Corona is
  single-threaded in the sense it just runs the coroutine in the thread where it
  was started. If you want more threads, you can, but Corona won't move
  coroutines from one thread to another by itself, since this kills Rusts
  `Sync`/`Send` protection.
* Pretend it is the only library you'll ever need. Use it where it makes sense,
  but use the other tools where it doesn't.


In other words, Corona tries to aim at the use case where you need to cope with
some asynchronous communication, but it is not the bottleneck of your
application. It helps get it out of the way with as little effort as possible
and concentrate at the interesting parts of your code.

## A case study

Let's look at some examples where Corona will help you and some where it'll just
make your problem even harder.

### The good case

Let's say you're writing a service that answers some queries. The service will
have *only* few hundreds or thousands of connected clients at a time. The query
is potentially large and needs to be parsed on the fly and each query needs some
more data to fetch and then a computation to be performed. The expensive part is
the computation, not the IO.

It is not a problem to have few thousands little stacks on today's server. The
fact the parsing can happen only once, on the fly, as the data come in, is a
plus. After the query arrives, the fetching is done with a library returning
futures (let's say Hyper). You join the fetching futures together (notion better
expressed by the future combinators than procedural code) and wait on them (in a
procedural manner). In another step, the computation is submitted to a
background thread pool (futures-cpupool) and the coroutine waits for the
completion. When it is done, it can proceed to send the answer back.

This avoids bunch of `and_then` calls, complicated error handling and bunch of
retries on the parsing of the data, without incurring large performance
penalties. The performance lies in the expensive computations, which can happen
in parallel in the thread pool.

### The bad case

A DNS resolver. Such a thing needs hundreds of thousands concurrent tasks. And
most of its job is asynchronous IO. Having a stack (which takes at least few
kilobytes of memory) for each is probably little bit too much. Furthermore,
you'll want to tune the IO code and having stack-full coroutines in your way
won't help much with that.

That means you'll have to pay the price for less convenient library, but that's
OK as the IO here *is* the bread and butter of the application and you want a
very fine control of it.

## Comparison with others

Of course, Corona isn't the only library dealing with asynchronous code. So,
let's have a look at few of the others and highlight the differences, so you
know which one to choose.

### Futures and Tokio

These are actually not competing with Corona. Corona builds on top of them, it
is their extension. You'll meet them when you use Corona, because it doesn't
even try to hide them.

However, comparing to purely futures-based code, Corona allows writing code in
more procedural and sequential way. It also allows „sneaking“ asynchronous code
through the stack (into callbacks), while all the stack above a future need to
be made of futures as well.

On the other hand, Corona takes more memory because of the stacks it allocates.
And some flows are better expressed as the combinators than procedurally (try
expressing you want to process at most `n` tasks in parallel until everything is
handled ‒ there's a
[combinator](https://docs.rs/futures/0.1.18/futures/stream/trait.Stream.html#method.buffered)
for that).

### [Futures-await](https://crates.io/crates/futures-await)

These are very similar in the async-await notation and the way they allow
juggling coroutines and writing procedural-looking code. There are some major
differences, though:

* Corona allows „sneaking“ the async code through the stack, while futures-await
  can suspend the coroutine only in its top-level function.
* Corona works with stable Rust, futures-await is an experiment that needs
  nightly Rust and it'll take some time to stabilize (hopefully this year).
* Futures-await is much cheaper memory-wise. Therefore, once it stabilizes, it
  probably should be preferred in places where it is possible. It'll still be
  possible to combine with Corona in places where it makes sense.
* Combining Futures-await with futures-cpupool allows doing M:N threading,
  spreading load across multiple threads. Corona is single-threaded.

### [May](https://crates.io/crates/may)

May is a port of Go's semantics into Rust. It does stack-full coroutines, like
Corona. However, there are major differences:

* Rust is inherently incompatible with stacks moving between threads, which is
  exactly what May does. You have to be *very careful* what is called from
  inside the coroutines to not invoke undefined behavior. And one of the big
  selling points of Rust is that you don't have to be very careful most of the
  time.
* May's approach is all or nothing ‒ it uses its own networking primitives that
  are not compatible with the rest of the Rust asynchronous libraries. Do you
  have a library that returns a future? Good for you, but you can't use it with
  May. Do you want to use unix sockets? May doesn't have them yet. Do you want
  to use something exotic, like `timerfd`, or use `recvmmsg` on a UDP socket?
  Tokio allows you to plug that in (with some effort), which means it can be
  used with Corona, but I haven't seen a way to do that with May.
* May's code looks *exactly* procedural, like you were just starting a thread
  instead of coroutine, while Corona forces you to explicitly annotate the
  places and objects that can suspend.

## Example

Enough talking. Let's show a bit of an example. Some of the classical examples
are available in the
[repository](https://github.com/vorner/corona/tree/master/examples). Here we
have just a small snippet:

```rust
// This brings few types into scope. But it also enriches all the futures and
// streams and sinks with additional methods.
use corona::prelude::*;
// This thing turns an async stream to sync one, with suspending the coroutine
// when waiting.
use corona::io::BlockingWrapper;
use tokio_core::net::TcpStream;
// Yes, this is still from the old tokio-core, we'll migrate soon.
use tokio_core::reactor::Core;

...
let (mut sender, receiver) = futures::unsync::mpsc::channel(10);
let mut core = Core::new()?;
let handle = core.handle();

let completed = Coroutine::with_defaults(handle.clone(), move || -> Result<(), Error> {
    // The connect returns a future.
    let stream = TcpStream::connect("127.0.0.1:8080".parse()?, &handle)
        // Will suspend the coroutine until it connects and return a resulting stream
        .coro_wait()?;

    let pseudosync_stream = BlockingWrapper::new(stream);
    // The StreamDeserializer knows nothing about async… but it doesn't matter.
    // The wrapper will just suspend the coroutine if it needs more data and the
    // parser will happily wait on the suspended stack, with all its internal
    // state.
    let bunch_of_jsons = serde_json::StreamDeserializer::new(pseudosync_stream);
    for json in bunch_of_jsons {
        // Handle parse errors. See, we can use ? in async code!
        let json = json?;
	// This just suspends the coroutine while the sink is full.
	// Note: this does *not* consume the sender while the send is in
	// progress, like the normal send does. You don't have to extract it
	// from the result again and assign it back to the variable.
        sender.coro_send(json)?;
    }
    Ok(())
});

// We could start another coroutine to handle the jsons here. Or bunch of other
// coroutines to do something completely different.

// We can use the coroutine as a future and wait for it.
// It will resolve to the result of the closure.
match core.run(completed) {
    Ok(()) => println!("Oh, good!"),
    Err(e) => println!("Oh no :-(: {}", e),
}
```

## The future

So, what happens with the library now? Tokio announced new version few days ago,
while Futures will do so in a short time. I'll let them settle and adapt the
code to the new API. I also plan to add scoped spawning support. Both of these
might or might not introduce breaking changes ‒ but not something substantial,
just added parameter here, or moving a method from a type to a new trait.

I have some planned functionality, but I'm pretty sure that one will be added
without any breakage. So that can go in as I implement it (or as someone else
helps me implement ‒ there's a `TODO` file in the repository, if you want to
help or want some functionality sooner).

But the long-term future is up to you, the user. My point of view of the API is
biased, as I'm the author. I want to hear from other people, what is easy to
use, what needs improvements. Pull requests are also welcome.
