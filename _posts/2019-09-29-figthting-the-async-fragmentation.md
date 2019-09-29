# Fighting the Async fragmentation

Sometimes, I get this nudging feeling that something is not exactly right and
that I have to go out and save the world and fix it (even though it's usually
something minor or doesn't need fixing at all). I guess everyone has days like
these. It's part what drives me to invest my free time to writing software.

This is about some dead ends when trying to fix the problem of Rust's async
networking fragmentation. I haven't been successful, but I can at least share
what I tried and discovered, maybe someone else is having the same bugging
feeling so they don't have to repeat them. Or just maybe some of the approaches
would work for some other problems. And because we have a bunch of success
stories out there, having some failure stories to balance it doesn't hurt.

## The problem

As I've mentioned in the [previous post](/2019/09/15/play-with-new-async.html), I'm
a bit worried about the fragmentation of the ecosystem. There are several
libraries out there, each one having its own `TcpListener` and threadpool and
timeouts, etc.

Imagine I'm writing a (very useful, of course) server that just tells everybody
to go away and I want to publish it as a crate for everyone to benefit from
that.

```rust
use std::io::Result;
use std::net::{IpAddr, Ipv4Addr};

use log::warn;
use tokio::net::{TcpListener};
use tokio::prelude::*;

pub async fn go_away_server(port: u16) -> Result<()> {
    let mut listener = TcpListener::bind((IpAddr::V4(Ipv4Addr::UNSPECIFIED), port)).await?;
    loop {
        match listener.accept().await {
            Ok((mut connection, _)) => {
                tokio::spawn(async move {
                    let _ignore_result = connection
                        .write_all(b"Go away, the Internet is full\n")
                        .await;
                });
            }
            Err(e) => warn!("Failed to get another connection: {}", e),
        }
    }
}
```

Now I have tied the user of my library to [`tokio`]. This has several
consequences. First, they have to run the tokio runtime, otherwise it won't work
(I suppose one could get away with running only *part* of the runtime and using
a different executor). Second, it's quite a heavy dependency. What if they
already use [`async-std`] or depend on some other library that uses
[`async-std`]. Certainly having both (and running part of runtime of each) is
wasteful on many fronts.

So, optimally, I would write my library in a way to enable either one or the
other (or both) by some cargo features or by passing a parameter or something
like that.

## Existing solution: The [`runtime`] crate

The crate provides a facade over other libraries (and creates another runtime of
its own, but it can be opted out from). The motivation and ideas are certainly in
the right direction. There are few disadvantages to it, though.

Its APIs are still somewhat limited. Many knobs are missing. It's quite unclear
what happens if I write some code (like the `go-away` crate) using this library
and then the user just runs the [`tokio`] runtime directly, instead of using the
`#[runtime::main]`.

There are some hints the library doesn't continue its development, at least for
now.

And additionally, the abstractions have runtime costs. Each socket adds another
heap allocation to a socket and a dynamic dispatch to each operation on the
socket. While this might not matter for 95% of use cases, someone will for sure
find themselves in the 5% (this is Rust, so people push the limits and do crazy
stuff with the tools they get). And even if the costs wouldn't matter in
practice, they have psychological effect and why not try solving the problem in
the hardmode? Long live the zero cost abstractions.

## Crazy idea: huge macro

I could basically implement the whole crate inside a macro. Then, depending on
feature flags it would instantiate variants of the code for the relevant back
end.

```rust
macro_rules! impl_go_away {
    ($listener: ty, $spawn: expr) => {
        pub async fn go_away_server(port: u16) -> Result<(), Error> {
            // The stuff...
        }
    }
}

#[cfg(feature = "async-std")]
pub mod async_std {
    impl_go_away(async_std::net::TcpListener, async_std::task::spawn);
}

#[cfg(feature = "tokio")]
pub mod tokio {
    impl_go_away(tokio::net::TcpListener, tokio::spawn);
}
```

This would work, but it doesn't seem exactly right. It's copy-pastish to the
point where the public API will be present twice, even in documentation. It
also doesn't really scale ‒ if there appears another networking library, support
for it needs to be added inside the `go-away` crate, it can't be added from
outside.

## The obvious idea: traits

This is exactly what traits were meant to do. Well, they were meant to do much
more, but this is definitely one of their use cases. If there's a million
`TcpStream` and `TcpListener` types around, all with the same methods, we'll
just create a trait that describes the interface. Then we'll just add another
trait, `Runtime` that'll have bunch of associated methods or types (so we don't
have to pass it around as a parameter) and we are done, right?

```rust
pub async fn go_away_server<R: Runtime = Autodetect>(port: u16) -> Result<()> {
    let mut listener = R::TcpListener::bind((Ipv4Addr::UNSPECIFIED, port))
        .await?;

    unimplemented!("...")
}
```

How would the trait look like?

```rust
pub trait Runtime {
    // Some network types to start with.
    type TcpListener: TcpListener;
    type TcpStream: TcpStream;
    type UdpSocket: UdpSocket;

    // We also need to be able to spawn stuff.
    fn spawn<F: Future<Output = ()> + Send + 'static>(f: F);
}
```

But wait, some libraries have spawns that return a `JoinHandle` ‒ basically a
future that returns the result once the passed-in future terminates. So, how do
we add it to our trait?

```rust
pub trait Runtime {
    // ...

    fn spawn_with_handle<R, F>(f: F) -> impl Future<Output = R>
    where
        R: Send + 'static,
        F: Future<Output = R> + Send + 'static;
}
```

**Bang!** This doesn't work. Traits don't like `impl Trait` in return position.
Oh, well, this is already solved problem, isn't it? We'll just add an associated
type and return that. It's been used for a long time this way.

```rust
pub trait Runtime {
    // ...

    type JoinHandle<T: Send>: Future<Output = T>;
    fn spawn_with_handle<R, F>(f: F) -> Self::JoinHandle<R>
    where
        R: Send + 'static,
        F: Future<Output = R> + Send + 'static;
}
```

**Bang!** Generic Associated Types. That's apparently an unstable feature. So
unstable the compiler will warn you it's going to crash when you enable it. And
then it delivers on its promise. Oh, well. Last desperate attempt:

```rust
pub trait Spawner<T> {
    type JoinHandle;
    fn spawn_with_handle<F: Future<Output = T> + Send + 'static>
        -> Self::JoinHandle;
}

pub trait Runtime {
    // ...

    type Spawner: for<T: Send + 'static> Spawner<T>;
    fn spawn<F: Future<Output = ()> + Send + 'static>(f: F) {
        Spawner::spawn_with_handle(f);
    }
    fn spawn_with_handle<R, F>(f: F) -> Spawner<R>::JoinHandle
    where
        R: Send + 'static,
        F: Future<Output = R> + Send + 'static
    {
        Spawner::spawn_with_handle(f)
    }
}
```

**Bang!** Higher Ranked Trait Bounds work only for lifetimes, not for types. We
can't do that `for<T: Send> ...`. We can move the trait bound to the `where`
clause of the `spawn_with_handle`, but then libraries will have to declare what
all types they'll want to spawn with. That's suboptimal, but workable.

Tokio doesn't seem to have a `JoinHandle`, but we can work around that using the
oneshot channel like this:

```rust
#[derive(Copy, Clone, Debug, Eq, PartialEq, Ord, PartialOrd, Hash)]
pub struct Tokio;

fn cancel_to_panic<T>(result: Result<T, Canceled>) -> T {
    result.expect("Joined task panicked")
}

type CancelToPanic<T> = fn(Result<T, Canceled>) -> T;

impl<T: Send + 'static> Spawner<T> for Tokio {
    type Handle = Map<Receiver<T>, CancelToPanic<T>>;
    fn spawn<F: Future<Output = T> + Send + 'static>(f: F) -> Self::Handle {
        let (sender, receiver) = oneshot::channel();
        tokio_executor::spawn(async move {
            let result = f.await;
            let _ = sender.send(result);
        });
        receiver.map(cancel_to_panic)
    }
}

impl Runtime for Tokio {
    // ...

    // Optimized version for handle-less spawning
    fn spawn<F: Future<Output = ()> + Send + 'static>(f: F) {
        tokio::spawn(f);
    }
}
```

This is a bit hairy because we have to name the concrete type of `Handle`, which
is a bit complex, but whatever.

## Traits, continuation

Let's look at the traits for the sockets now. How does a `TcpStream` look like?
Well, we can read and write it, there are some methods to create it and some
methods to manipulate it.

```rust
pub trait TcpStream: AsyncRead + AsyncWrite {
    async fn connect(addr: SocketAddr) -> Result<Self>;
    async fn pair() -> Result<(Self, Self)>;
    fn local_addr(&self) -> Result<SocketAddr>;
    fn peer_addr(&self) -> Result<SocketAddr>;
    fn shutdown(&self, how: Shutdown) -> Result<()>;
}
```

**Bang!** `async fn f() -> T` is the same thing as `fn f() -> impl Future<Output
= T>` for all the external observers. Therefore, they don't work inside traits.
There's this [`async-trait`] crate that should help with that, but it converts
them to `fn f() -> Box<dyn Future<Output = >>` and we said we don't want boxing
and dynamic dispatch.

So, once again going with an associated type:

```rust
pub trait TcpStream: AsyncRead + AsyncWrite {
    type Connect: Future<Output = Result<Self>> + Send + 'static;
    fn connect(addr: SocketAddr) -> Self::Connect;
    // ...
}
```

Cool, let's implement it. Huh, but what would the `Connect` type be? Looking at
the
[`connect`](https://docs.rs/tokio/0.2.0-alpha.4/tokio/net/struct.TcpStream.html#method.connect)
in [`tokio`], we're doomed. It doesn't provide a named future, it returns `impl
Future`. So we can't really do the same trick we did with `Spawner` and create
an ugly but concrete type by wrapping future combinators together. Too bad.

There's this `type_alias_impl_trait` unstable feature which would allow us doing
it this way.

```rust
#![feature(type_alias_impl_trait)]

// ...

impl TcpStream for tokio::net::TcpStream {
    type Connect = impl Future<Output = Result<Self>> + Send + 'static;
    fn connect(addr: SocketAddr) -> Self::Connect {
        tokio::net::TcpStream::connect(addr)
    }
}
```

This works, but it can be compiled only with the nightly compiler. Which
doesn't help me, because I actually wanted to use the abstraction layer in some
of the crates I maintain, and they aim at stable Rust. One can hope this'll get
stabilized soon (it's this
[tracking issue](https://github.com/rust-lang/rust/issues/63063)), it seems to
be working alright as far as I've tried. And this'll be getting more and more
important, as the `async/await` notation creates unnameable types. One could
create `impl Future` types before too, but at least most of the low level
libraries had concrete types for mostly everything. Not any more.

## Traits, the Fn hack

There's one loophole how to refer to such unnameable return type in stable Rust.
That's the `Fn` trait and its `Output`, in about this way:

```rust
trait Creator<T> {
    type Fut: Future<Output = std::io::Result<T>> + Send + 'static;
    fn create(&self, addr: SocketAddr) -> Self::Fut;
}

impl<F, Fut, T> Creator<T> for F
where
    F: Fn(SocketAddr) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = std::io::Result<T>> + Send + 'static
{
    type Fut = Fut;
    fn create(&self, addr: SocketAddr) -> Self::Fut {
        (*self)(addr)
    }
}
```

Then, if we hold the type of such creator function, we can let Rust do its magic
of type inference and steal the type from the associated type. We can declare
that `TcpSocket::connect` is our `Creator`.

The problem is, the `TcpSocket::connect` is an expression, not a type. We need
its type and it's unnameable as well. Therefore, *we* need to be generic over
the creator and our instance be created with `TcpSocket::connect` passed to it,
so our own type is deduced.

This basically means that we need to store the function and pass the Runtime as
value, not as type parameter.

```rust
fn go_away_server<R: Runtime>(runtime: &R, port: u16) -> async Result<()> {
    let mut listener = runtime.bind_listener(...).await?;
    ...
}
```

This is possible but it looks wrong. Furthermore, with the type parameter we
could place it on a whole struct and have it available in the whole
implementation of the struct, now we have to keep passing the argument around to
all the methods or store it.

## Closing thoughts

The traits feel like the correct direction in general. However, with the
limitations as there are now, it feels like navigating a mine field. One one
hand, I hope some of the limitations will get lifted eventually. On the other
hand, I have to ask where this will end. I mean, every time some new shiny
feature is added to the language, some more that are obviously missing pop up as
a result. Is there a „closure“ of the language? Some set of features no missing
ones keep sticking out of it?

As for the library fragmentation, I hope some solution will be found eventually.
My naïve wish would be for all the libraries to agree with each other, take
their socket types (and whatever wakes them up) and put it into a common crate
and take it into each as a dependency. Then `tokio::net::TcpStream` would just
happen to be the same type as `async-std::net::TcpStream`. After all, their
interface is almost the same and the thing that wakes them (is it called
driver?) is mostly just a wrapper around [`mio`]. But maybe there are good
reasons why this doesn't work.

[`Future`]: https://doc.rust-lang.org/std/future/trait.Future.html
[`tokio`]: https://crates.io/crates/tokio/0.2.0-alpha.5
[`runtime`]: https://crates.io/crates/runtime/0.3.0-alpha.7
[`async-std`]: https://crates.io/crates/async-std
[`async-trait`]: https://crates.io/crates/async-trait
[`mio`]: https://crates.io/crates/mio
