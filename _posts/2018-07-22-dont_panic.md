# Don't panic

Rust offers guarantees about memory safety and data race freedom. But it doesn't
stop there. It nudges the programmer in the general direction of good habits.
For example, the `Result` is designed in a way it is easier to handle the error
than ignore it and even ignoring it must be a conscious choice. This helps build
software that is robust and with lower concentration of bugs.

Designing API in this way, in a way that the path of least resistance is the
correct one, is often called „[The Pit of Success]“. At its most basic form,
this is about choosing the right default behaviours. No matter what your use
case is, the default choice does not have to be the best possible, but it
must not be outright *wrong*. If you don't read the documentation, you should
stumble upon something that doesn't have hidden gotchas. Something that doesn't
compile is fine, but something that seems to work in 99% of cases, but kills a
kitten in the one remaining percent is not.

And there's one particular area where I believe Rust falls pretty short to do
that. I can guess at the reasons why it ended up as it is, which I actually do
at the end of this article.

The area is handling situations the program doesn't know how to handle ‒
panicking.

## What is that panic thing anyway?

Rust comes with different ways to handle error conditions. In its simplest, it
divides the errors in two groups (technically, into three, but the third one is
for things where you no longer even have a viable definition of sanity, so the
program simply gives up and I'm not going to talk about that).

The first one is for things the program knows how to cope with. The programmer
foresaw that you could lack disk space, or that the program could be given
invalid input by the user. In the ideal world where the programmer writes only
correct programs, this includes everything the program encounters during its
lifetime. Rust handles these errors mostly through the `Result` type.

The other kind is for things the program *doesn't* know how to handle. That, in
some definition of correct programs, means *bugs*. Either something inside the
program broke, or the programmer forgot to take some real situation into
account.

Admitting all code contains bugs and therefore it makes sense to have special
language support for handling bugs is definitely a very enlightened approach.

Rust has the concept of panic to handle the bugs. Every time you index out of
an array, an `assert` fails, or an `unreachable!()` piece of code is executed, a
panic happens.

By default in such case the runtime starts unwinding the stack (removing
functions one by one) of the current thread, calling destructors of things on
the stack, until [`catch_unwind`] is reached or until the whole stack is
unwound, in which case the thread terminates. However, if it was not the main
thread, the show goes on.

## Why is this a problem?

First, panicking is non-local and invisible flow control. Mostly anything can
panic. Therefore, mostly nobody takes that into account when writing code and a
panic can lead to leaving things in an inconsistent state. This is *usually*
fine, because everything in that one thread dies anyway. Except when this is not
fine, like when your `drop` gets confused by the inconsistent state, when
something doesn't die because it is shared or when you play around with `unsafe`
code. Yep, be extra careful when combining callbacks and `unsafe`. Like, even
*more* extra careful than when handling just `unsafe`. Because that callback can
panic and then the rest of your `unsafe` function won't run. By the way,
`catch_unwind` (or `panic` itself) can allocate, and there are situations when
you can't afford to allocate (inside signal handlers, when you play with `fork`,
if you are the allocator itself), so there's no reasonable way to panic safely.

Panicking *through* some things is not safe. For example, through an FFI
boundary. So if you pass a rust callback into some C function (or a function
that pretends to be a C function), such callback must never panic. But, well,
most anything non-trivial in Rust can panic, because panic is for handling bugs,
and anything can contain bugs. So you have to somehow wrap the callback and deal
with the possible panic, but it is all too easy to forget (if you do panic
across FFI boundary, Rust will abort the program, but that was not always the
case).

In all these cases, you either forget and magnify the bug, or you pay the tax by
making your code more complex (and run the risk of introducing more bugs because
of the complexity).

Furthermore, if you start a thread to perform some work ‒ to clean up some
things in the background, to do some work and tell you when it is done through a
channel, or bunch of threads that all synchronize with each other through a
[`Barrier`] and that thread dies with panic, you get an application in
an inconsistent state. The application runs on, but without the thread to do the
cleanups, or the work is never finished and the main coordinating thread is
never notified or all the other threads wait forever for the one that went belly
up. And you aren't likely to see anything in the logs either, because unless you
register your own panic hook, it prints a message to `stderr`, not to where you
currently log (again, you can change that with for example [`log_panics`], but
you must not forget ‒ and you need to know in the first place that you must not
forget).

This is obviously wrong ‒ having an application in a half-dead state in
production, but not getting it restarted, for who knows how long, is something a
robust application doesn't do. You may have noticed this virtually never happens
to C or C++ programs (because C can only `abort` and unhandled exception in C++
also aborts), but very often does to Java ones. An exception backtrace scrolls
through the screen when you start it, and the application sits there not doing
what it should, in some zombie state ‒ not alive, but not dead either.

As an exercise, try to come up with a way to fix each of these (each one is
the natural way to write the thing and each one is wrong around panics),
preferably with only standard library primitives:

```rust
fn main() {
    thread::spawn(|| {
        loop {
            thread::sleep(Duration::from_secs(3600));
            // If this panics, we never get any cleanup done, eating
	    // resources forever. Bad :-(
            do_periodic_cleanup();
        }
    });
    while (keep_running()) {
        let request = receive_request();
        let response = do_computation(request);
        send_response(response);
    }
}
```

```rust
fn main() {
    let (sender, receiver) = mpsc::channel();
    let sender_a = sender.clone();
    let sender_b = sender;
    // If either of these panics, only the other things will get
    // generated from then on, ignoring the others.
    thread::spawn(move || produce_as(sender_a));
    thread::spawn(move || produce_bs(sender_b));
    for a_or_b in receiver {
        consume(a_or_b);
    }
}
```

```rust
fn main() {
    let cpus = num_cpus::get();
    let barrier = Arc::new(Barrier::new(cpus));
    let mut threads = Vec::new();
    for _ in 0..cpus {
        let barrier = Arc::clone(&barrier);
        // If one thread (other than the first) panics in
	// preparation_phase or real_computation, the program
	// never finishes.
        threads.push(thread::spawn(move || {
            preparation_phase();
            barrier.wait();
            real_computation();
            barrier.wait();
            cleanup();
        }));
    }
    for thread in threads {
        // This would nicely propagate the panic… if it wouldn't
	// deadlock on the first thread.
        thread.join().unwrap();
    }
}
```

Also, you pay the price for being able to panic even though you never do.
Functions that contain at least one value with a destructor (even a generated
one) need to create „landing pads“ ‒ markers on the stack that are used during
the unwind. These don't come completely free, so this is against the philosophy
of paying only for what you're going to use. These costs are somewhat defensible
in C++, where exceptions are commonplace, but panics should not happen in a
correct, production program *at all*.

As a side note, I wonder if one could implement unwinding using debug info and
some global lookup table of what destructors to call in a function depending on
the position in the function ‒ that would be probably significantly more
expensive to unwind, but the only cost when not unwinding would be including the
table in the program binary).

## Possible solutions

If there was a time machine that could send a message back in time, I think the
right solution would have been to implement `panic` as always aborting the whole
program, full-stop. Or with a hook, but the default definitely being to abort.
(and renaming it to a more descriptive `bug`). In general, it is a pretty
stupid idea to try to do a clean shutdown if the program is in an inconsistent
state and you have to be able to cope with unclean shutdown anyway (OOM killer,
cut power, …). All service managers handle programs that died, but you need to
have your own code to check the health of your program if it can end up in an
undead state.

What you can do now, though, is setting this in every binary crate you write:

```toml
[profile.dev]
panic = 'abort'

[profile.release]
panic = 'abort'
```

This way your program will die instead of staying in an undead state, saving the
costs for exorcism in your server dungeon. This doesn't help you if you are a
library author, and the standard library still places the unwind landing pads,
though.

I don't know if it would make sense, but `cargo new` could actually put this
into newly created projects. Changing the default behavior for already existing
applications that didn't specify panic strategy is probably not going to cut it
due to backwards compatibility.

Some help could be having synchronization primitives that offer a reasonable way
to cope with panicking threads. A `Mutex` can [get poisoned] if it was locked
while panicking and anyone else touching it will then get an error (which is
commonly handled with `unwrap`, propagating the panic). Furthermore, having a
thread manager that terminates with an error when any of the managed threads
dies would help, as would a channel that would include notifications about dead
senders to the receiver or a barrier that waits for as many threads as there are
handles (a dead thread would drop its handle). If anyone has too much free time
and doesn't know how to go about it, I'm willing to mentor the work.

## What lead to this state?

I believe the biggest reason is that Rust changed its core philosophy before
reaching 1.0. It stared as something very similar to how Go looks like, or how
Erlang looks like. It had green threads and panicking was supposed to be
often-used error handling mechanism.

This paradigm was dropped (AFAIK because there was no reason to have two Gos
just with different names), but some traces are still visible ‒ panics being one
of them.

[The Pit of Success]: https://blogs.msdn.microsoft.com/brada/2003/10/02/the-pit-of-success/
[`catch_unwind`]: https://doc.rust-lang.org/std/panic/fn.catch_unwind.html
[`Barrier`]: https://doc.rust-lang.org/stable/std/sync/struct.Barrier.html
[`log_panics`]: https://crates.io/crates/log-panics
[get poisoned]: https://doc.rust-lang.org/std/sync/struct.Mutex.html#poisoning
