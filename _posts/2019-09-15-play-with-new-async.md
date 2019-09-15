# Playing with the new async

I'm quite a fan of the asynchronous programming styles (not spawning too many
threads, using some kind of [epoll] instead to coordinate what to do next). I've
written few event loops manually in C. My first [Rust project][eveboros] was an
attempt to write an event loop (luckily, I've never released that one; while it
did work, the API turned out just terrible). I've done some development with old
[tokio-core], the bit newer [tokio] and wrote a [coroutine library][corona] on
top of [tokio] as an attempt to make the working with async in Rust more
convenient while not having to wait for the full compiler support.

So, the Rust's async-await is an area of interest for me. However, due to all
the churn around it, I was mostly staying with the tokio 0.1 and waiting it out
to see how everything develops.

But I've decided it's about time to have a closer look at the new stuff and
play with it a bit. Partly because I feel I want to see it before it is released
„for good“ (eg. in stable 1.39), but partly because I wanted to measure some
network throughputs, but in a different way than [iperf3] does (well, you can
call it an excuse to play with Rust over the weekend instead of reason, if you
want). So I've decided to write my own tool and see if the async-await and all
the libraries around it are up to it. And I finally found out a bit of time. The
result can be found [here][tspam].

## The task

In my job, I work with kind of a [smart firewall thing][omni] that sits in an
embedded device, small enough to be put to users' homes. And while we have some
performance measurements, these mostly concentrate at not doing visible impact
to normal home traffic usage.

But the thing with benchmarking is, you never run out of ideas what to measure.

The question I'm asking right now is how many new connections per second can
I open before the device starts sweating. We know the number is large enough to
not happen during normal browsing, but we don't know the exact number yet.

So I've decided to write this tool that'll keep opening many new connections and
sending only little data over each. Then it closes the connection and measures
the latency of the whole exchange. That's easy in principle, except I'm
dry-running it on a gigabit ethernet link without anything in between the two
computers and I want the program to keep up with that too 😈. Just to make sure
I *can* give the device enough traffic to start sweating.

While simple, having configurable of data to be sent in each direction also
allows me to test in many scenarios ‒ empty connections, connections with some
unrecognized content, big ones, small ones, http connections (this can be faked
by just placing the request and response into a file and sending them), etc.

## Initial research

I knew about the async-await notation and that I need nightly to compile that.
The question was, what libraries are out there to provide the actual primitives
to await on. Luckily, I wanted only the basic networking stuff ‒ only TCP
connections, no fancy protocols on top, no encryption.

I've discovered three libraries that provide that:

* [tokio] (in its alpha releases of version 0.2)
* [async-std]
* [runtime]

This was both good and bad news. On one side, it means there are multiple
projects working on the problem and therefore there's life around the problem.
There must be something that'll fit my needs in that, right? The community must
be thriving if it can feed three concurrent implementations.

On the other hand, there's one synchronous [TcpStream][std-tcpstream] in the
standard library. If I'm counting the [tokio-v0.1][tokio] and [tokio-core],
there are 5 pairwise-incompatible asynchronous `TcpStream`s. Libraries built on
top of these will either need to be generic, using traits, or will depend on
specific one of them. If you happen to need two in the same program, you have to
bring both dependencies in. Furthermore, it feels like duplicating the effort
and that there'll be further time needed before the dust settles and
compatibility layers appear ☹. It'll mean triple the amount to work to bring
the async support for [signal-hook] up to date.

Anyway, back to my problem of choosing the library. The [runtime] went off list
pretty fast. The library seems to be great for examples and tutorials, but seems
to be lacking some kind of flexibility to meet real-world needs. In particular,
I wasn't able to find a way to have only *part* of my program asynchronous. The
library is all or nothing ‒ the whole `main` must be marked as
`#[runtime::main]` (and I can't place the attribute at differently named
function). However, I wanted to do a synchronous startup, then run something
asynchronous, wait for it to finish. On the client side, I actually wanted to
have multiple *rounds* of this. I probably could invest some mental effort and
shoehorn my idea of what I wanted to do into all-async code, but I didn't want
to. After all, some real world projects I've worked on do have some synchronous
threads and some asynchronous ones. I wanted to be able to switch between the
modes and not being able to feels limiting.

I've decided to go with [async-std], for two reasons. First, it tries to reflect
the `std` API as closely as possible, which I find a nice idea. Second, I
already worked with [tokio], so I was curious about the alternatives.

## The first implementation

I know the infrastructure around the `std::future` and such is young. I expected
problems to appear.

However, the initial implementation went much smoother than expected. Due to the
familiarity of the APIs I could write most of the code from memory, only with
occasional check and glimpse at an example how to put some bits together. I had
both the server and the client ready in few hours (with most of the bells and
whistles like command line parsing with help, etc; did anyone say Rust is not
good for prototyping?). The only thing I've noticed was that [async-std] didn't
contain a channel, which in my experience is quite often used tool in many
applications that also need async. This wasn't a big problem, since the
[futures-preview] is already a dependency of [async-std] and it provides one.

But when running it, with both server and client on `localhost`, sometimes the
experiment never finished, even with low rates of new connection creation. This
seemed odd. I mean, on a real network one can expect a packet to get lost, but
it gets retransmitted and the problem should resolve itself in a short
time. Furthermore, this was a `localhost`, not a real network. Maybe I was
doing something wrong?

## Timeouts

Nevermind, I wanted an easy way out. So I've decided to add a timeout to each
connection and get rid of the broken ones. Or to prove the problem was somewhere
else.

OK, there's the [timeout] function. But it says it is supported only on
`unstable` ‒ what does that mean and how do I turn it on? The docs don't say.
Trying to use it didn't provoke the compiler to issue an useful hint either.

But after using the old futures, I could do such thing by hand, of course.
One only needs to create an independent time-based future and select which one
happens first. Dropping the other will cancel it. [async-std] offers such
future, it's called [sleep].

And a roadblock appeared. All kinds of `select` things expect the futures to be
[Unpin]. My connection future wasn't. That wasn't too unexpected. What
surprised me that even the (probably quite stateless) [sleep] wasn't. Maybe
some basic futures provided by the library eventually will become [Unpin], to
make them more widely usable.

The problem can be solved, and quite easily *once the concept of pinning is
absorbed*. There's the [`pin_mut`] macro which will take care of everything
needed. But even though I watched the development of async-await a bit and knew
about pins, I had to spend some time searching for the right incantations.
Previously, selecting over two futures was a no-brainer, currently it poses
additional friction. The code is not really short either (I've preferred the
function over the macro version to avoid rightward drift of `match` in
`match`-like macro).

```rust
async fn connect(server: SocketAddr, content: Arc<[u8]>, mut results: Sender<Duration>) {
    let connect = connect_inner(server, content);
    pin_mut!(connect);
    let timeout = task::sleep(TIMEOUT);
    pin_mut!(timeout);
    match future::select(connect, timeout).await {
        Either::Left((Ok(duration), _)) => results
            .send(duration)
            .await
            .expect("Channel prematurely closed"),
        Either::Left((Err(e), _)) => error!("Connection failed: {}", e),
        Either::Right(_) => {
            warn!("Connection timed out");
            results
                .send(TIMEOUT)
                .await
                .expect("Channel prematurely closed");
        }
    }
}
```

## Switching to tokio

I've used 5 seconds timeout. That is an *ethernity* when it comes to solving
some connectivity issues on *localhost*. Still, even with rates like 100 new
connections per second, some of the connections were timing out. But only very
few of them, so it didn't look like I was doing something terribly wrong
myself. I suspect the [async-std] is missing some wakeup registration in one of
its primitives. I'll try to pin the problem down further and report the bug
later on, but I want to have more data for that first (bugreports in the style
of „*Something* doesn't work as expected are *not* useful).

Anyway, I wasn't even *sure* this was the fault of the library. So I've decided
to try switching to [tokio] and see if it goes away.

The API is mostly similar, with some exceptions. For example, there's no
free-standing `block_on` function. It seems `tokio::run` is gone too from the
0.2 branch ☹. There's the `[tokio::main]` attribute, but once again, I don't
want my *whole* program to be asynchronous. Luckily, I can build a [Runtime]
manually and that one *does* have `block_on` (did the old Runtime had some more
tweaking knobs, though?). It even allows me to reuse the runtime between the
multiple asynchronous parts, something I'm not sure the previous free standing
`block_on` of [async-std] did (maybe it did, but the docs don't say much about
what happens under the hood; It doesn't allow much room for tweaking too, but
maybe the crate is just too young and there'll be a way in the future).

The other part I was missing was an async port of the `std::io` module. I
wanted to read and throw away all the data from the connection, which can be
done in the [async-std] easily:

```rust
io::copy(&mut connection, &mut io::sink()).await?;
```

Tokio has a `.copy` method on the TCP connection (this might be coming from
[futures-preview], actually), but no blackhole to write into ☹. Writing one is
quite straight-forward, but still the amount of code feels like it's more
than the problem deserves. I guess I'll just post a pull request to either
[futures-preview] or [tokio], someone might find it useful (with a read
counterpart that's always empty).

```rust
struct Sink;

impl AsyncWrite for Sink {
    fn poll_write(
        self: Pin<&mut Self>,
        _: &mut Context,
        buf: &[u8],
    ) -> Poll<Result<usize, IoError>> {
        Poll::Ready(Ok(buf.len()))
    }
    fn poll_flush(self: Pin<&mut Self>, _: &mut Context) -> Poll<Result<(), IoError>> {
        Poll::Ready(Ok(()))
    }
    fn poll_shutdown(self: Pin<&mut Self>, _: &mut Context) -> Poll<Result<(), IoError>> {
        Poll::Ready(Ok(()))
    }
}
```

*Edit: It turn out this is already in master of [futures-preview], just not yet
released.*

However, when I run it again after the switch, the experiment was running for
much longer than it should have. At that time the hardcoded time of one run was
10 seconds (plus waiting for all the connections to finish, but the maxima were
in terms of milliseconds). It felt much longer.

The problem turned out to be tokio's [timer] inaccuracy (which is a feature). I
always started a new connection and then computed the new time at which I'd
like the next one to start by adding a pause interval. However, the timer's
accuracy is one millisecond and with rates over 1000 connections per second,
this was not enough, it always slept for that one millisecond. This was an easy
fix, just changing from adding one „pause“ between connections to the previous
time, I count `n` pauses from the base time. This has the slight disadvantage
of of „batching“ the connections, though ‒ if I run at a rate of 25k, it
creates 25 connections and then sleeps a millisecond, because all 25 times
round to the same millisecond.

Anyway, the timeouts went away. As a bonus, the program seemed to use slightly
less CPU, but I didn't do much scientific measurement of that, I've simply
eyeballed `top` ‒ so it might as well an impression or the change of sleeping
algorithm (executing in batches could mean less context switching or something).

## Performance tuning

Actually getting to a point when the connections and data sent over them
saturates the gigabit ethernet (connecting my desktop and laptop, both quite
ordinary machines) took some effort. Surprisingly, most of it was tuning of the
*kernel* and OS, not of the code in the program. The biggest problem was
running out of client source address/port pairs (see the [repo][tspam] for
the details).

I've done *some* tweaks in the code too, like having multiple sockets accepting
on the same port, running in independent tasks (therefore able to run in
parallel threads) or tuning the listen queue length. I might eventually add
support for multiple source IPs too.

Even with all the tuning, at high rates (tens of thousands of new connections
per second), the behaviour is a bit glitchy ‒ sometimes, connections are reset
by either side. Sometimes, on the client, something in the kernel goes mad and
starts eating a lot of (all of the) CPU while dropping performance
significantly. So there is still space for more research. I'm probably not
investing the time, though, for the purpose of the tool these rates are likely
several orders of magnitude higher than needed.

And I've noticed one weird thing. With growing rate, the mean latency tends to
*shring*. Well, up to a point. But it seems the thread pools are faster when
they are kept a bit busy.

## Conclusions

I know I'm a picky person and I find problems wherever I look. So I'm not
surprised to have found them here. Actually, my feeling about the state of the
async-await is quite positive. It's definitely more comfortable to use than the
original combinators way, though I still feel like the combinators are not
completely dead ‒ some idioms are simply easier to express with them, eg
[`buffer_unordered`].

Obviously, many things need some more love and polishing ‒ I've seen
dead links in documentation, functionality that I was used to but went away,
bugs… but that's to be expected at this state. The stuff I used did say
„alpha“.

As mentioned above, I'm a bit worried about the schism of libraries.

I'm quite impressed in how little work it took to write a high-performance
implementation. I mean, if the first part that gives performance-wise is the
kernel, then Rust has nothing to be ashamed of.

And, by the way, I haven't yet measured the device yet, only spent part of the
weekend playing with Rust.

[epoll]: http://man7.org/linux/man-pages/man7/epoll.7.html
[eveboros]: https://github.com/vorner/eveboros
[tokio-core]: https://crates.io/crates/tokio-core
[tokio]: https://crates.io/crates/tokio
[corona]: https://crates.io/crates/corona
[tspam]: https://github.com/vorner/tspam
[iperf3]: https://github.com/esnet/iperf
[omni]: https://www.avast.com/en-us/omni#omni
[futures-preview]: https://crates.io/crates/futures-preview
[timeout]: https://docs.rs/async-std/0.99.5/async_std/future/fn.timeout.html
[sleep]: https://docs.rs/async-std/0.99.5/async_std/task/fn.sleep.html
[Unpin]: https://doc.rust-lang.org/std/marker/trait.Unpin.html
[timer]: https://docs.rs/tokio-timer/0.3.0-alpha.4/tokio_timer/timer/struct.Timer.html
[`buffer_unordered`]: https://docs.rs/futures-preview/0.3.0-alpha.18/futures/stream/trait.StreamExt.html#method.buffer_unordered
[Runtime]: https://docs.rs/tokio/0.2.0-alpha.4/tokio/runtime/struct.Runtime.html
[async-std]: https://crates.io/crates/async-std
[std-tcpstream]: https://doc.rust-lang.org/std/net/struct.TcpStream.html
[signal-hook]: https://crates.io/crates/signal-hook
[`pin_mut`]: https://docs.rs/futures-preview/0.3.0-alpha.18/futures/macro.pin_mut.html
