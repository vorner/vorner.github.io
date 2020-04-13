# Hyper traps

As a responsible Rust Evangelist, I try to propagate the language in my current
job. So, once again, I've done a workshop, this time themed about async Rust.
The goal was to write a small but non-trivial http service.

As an aside, I tend to like the new working from home and most everything is
fine. But doing hands-on workshops is definitely easier in person than over
video-conferencing.

I didn't want to dig into specific http frameworks, because then I wouldn't be
teaching so much Rust, but the frameworks instead. So we went with using [hyper]
directly. In my experience, while a bit low-level, it's a good-enough tool when
you need to expose an API other stuff talks to over http, but don't need any
kind of HTML templating stuff. That's not to say a framework won't make your
life easier if you do a lot of complex http-related stuff, we simply went with
the smallest reasonable cannon that gets the job done.

And while the exercise went mostly well and both Rust and `hyper` are perceived
in a tentatively-positive way, there were two traps that tripped my colleagues.
I'd like to share them here ‒ there might be something that can be done about
them and even if not, it might save someone from the same traps.

*Note: I haven't verified each of these snippets compiles ‒ there might be typos
and such. They are illustrative.*

## Panics

While my stand on panics is that they are not supposed to happen and they are
Rust's bug-coping strategy, so they have no place at all in production
application, they do happen during development.

When a panic happens in the response handler, `hyper` (or maybe [tokio] that
sits below `hyper`) recovers ‒ it saves the worker thread, it kills just that
one task (connection) and the service can continue. Nevertheless, it does so in
a very minimal way. The client connection is just abruptly cut off. One would
like to have a 500-error response instead.

```rust
async fn handle(_: Request<Body>) -> Result<Response<Body>, Infallible> {
    unimplemented!()
}

#[tokio::main]
async fn main() {
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));

    let make_svc = make_service_fn(|_conn| async {
        Ok::<_, Infallible>(service_fn(handle))
    });

    let server = Server::bind(&addr).serve(make_svc);

    if let Err(e) = server.await {
        error!("server error: {}", e);
    }
}
```

It makes sense for `hyper` not to do that, it's a http library, not a framework.
It shouldn't have an opinion how your error page looks like, how much info it
contains and such.  So let's do that ourselves. With a little bit digging, one
finds the [`catch_unwind` method in `futures`](https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html#method.catch_unwind).

Few important things are:

* One wants to use this method, not the [`catch_unwind` from
  `std`](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html). This is
  because futures are lazy and we want to catch the unwinds from its `poll`
  method, not from the future being created (which, if you use an `async fn`,
  probably can't even panic).
* Your future is likely not going to be unwind-safe. Honestly, unwind safety in
  Rust is a bit weird concept. Anyway, as we will let the task die the same way
  it does now, we just provide the handy 500-error response to the client. So we
  are going to declare it unwind safe and be done with it.

```rust
type HttpResult = Result<Response<Body>, Infallible>;

async fn handle(_: Request<Body>) -> HttpResult {
    unimplemented!()
}

async fn handle_panics(fut: impl Future<Output = HttpResult>)
    -> HttpResult
{
    // 1. Wrap the future in AssertUnwindSafe, to make the compiler happy
    //    and allow us doing this. The wrapper also implements `Future`
    //    and delegates `poll` inside.
    // 2. Turn panics falling out of the `poll` into errors. Note that we
    //    get `Result<Result<_, _>, _>` thing here.
    let wrapped = AssertUnwindSafe(fut).catch_unwind();
    match wrapped.await {
        // Here we unwrap just the outer panic-induced `Result`, returning
        // the inner `Result`
        Ok(response) => response,
        Err(_panic) => {
            error!("A panic happened. Run to the hills");
            let error = Response::builder()
                .status(StatusCode::INTERNAL_SERVER_ERROR)
                .body("We screwed up, sorry!".into())
                .unwrap();
            Ok(error)
        }
    }
}

#[tokio::main]
async fn main() {
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));

    let make_svc = make_service_fn(|_conn| async {
        // Run `handle` immediately, which produces a Future (that's what
        // async functions do ‒ they return a Future right away, but don't
        // run that Future.). Feed that future to `handle_panics` to wrap
        // it up and return a panic-handling future.
        Ok::<_, Infallible>(service_fn(|req| handle_panics(handle(req))))
    });

    let server = Server::bind(&addr).serve(make_svc);

    if let Err(e) = server.await {
        error!("server error: {}", e);
    }
}
```

## Ownership & async blocks

Let's say the service grows a little bit in complexity and we want to extract
the handling into a separate struct. Something like this:

```rust
struct Handler {
    // Interesting stuff goes in here
}

impl Handler {
    async fn handle(_: Request<Body>) -> HttpResult {
        unimplemented!()
    }
}
```

The `Handler` should effectively be a singleton ‒ it contains some shared
information (maybe a handle to a database connection pool, or some global
configuration). So we could place it into a global static like this:

```rust
static HANDLER: Lazy<Handler> = Lazy::new(|| Handler::create());

    // inside main:
    let make_svc = make_service_fn(|_conn| async {
        Ok::<_, Infallible>(service_fn(|req| HANDLER.handle(req)))
    });
```

Nevertheless, this feels ugly because of the global variable. It's against
common best practices, complicates testing, etc. So we would prefer to create
one in main, wrap it in `Arc` and share between the handlers, something like
this:

```rust
let handler = Arc::new(Handler::create());

let make_svc = make_service_fn(|_conn| async {
    let handler = Arc::clone(&handler);

    Ok::<_, Infallible>(service_fn(|req| async {
        let handler = Arc::clone(&handler);
        Ok::<_, Infallible>(router.route(req).await)
    }))
});
```

This will, however, start throwing variations of „cannot move out of“ and
„does not live long enough“ lifetime errors. The usual tricks ‒ introducing more
`Arc::clone`s and placing the `move` keyword at various places doesn't help.

Now, the correct solution here is to move the `Arc::clone` *outside* of the
`async` block. I'll explain why that is in a while.

```rust
let handler = Arc::new(Handler::create());

let make_svc = make_service_fn(|_conn| {
    let handler = Arc::clone(&handler);

    async {
        Ok::<_, Infallible>(service_fn(move |req| {
            let handler = Arc::clone(&handler);

            async move {
                Ok::<_, Infallible>(router.route(req).await)
            }
        }))
    }
});
```

But for the explanation, we need to go a bit deeper.

### Independent lifetimes

Because the created service handlers and the futures need to be independent in
their lifetimes (we can't predict how long each one will live, because we don't
know how long the clients will keep their connections open), they need to be
`'static`. That means one closure or future can't just borrow from the parent
context. Therefore, we need the `move` keywords at strategic positions.

### We can't leave the parent context „hollow“

So we know the innermost `async` block needs to `move`. But if we just do that,
we consume the `Arc` from the closure. Yet, we can't do that because the closure
can potentially run multiple times ‒ creating multiple instances of the future.
If the first one consumes the `Arc`, what would be left for the second one?
(This is encoded in the closure being `FnMut`)

Therefore, we need to clone the `Arc` every time, so each future gets its own.

### The async blocks are lazy

The `async` block doesn't start running right away. It acts similar way to
closures ‒ it just produces an anonymous struct that implements the `Future`
trait. All the needed captured variables are stored in there, it's shipped off
and returned. It starts running only once the executor calls `poll` for he first
time.

Therefore, cloning the `Arc` inside the block, like in the first example, is
*too late*. By that time the block starts running, we have either moved (and
consumed) the `Arc` from the closure's context (and none is left for its second
run) or we stored only a reference (and the parent closure might have been
dropped by that time, creating a dangling reference), depending on if we use
`async move` or `async` only.

Therefore, we first have to clone our `Arc` and only then build the future. That
leads to having half of the closure body be synchronous/blocking, the other half
`async`.

### Solution: async closures

As you can guess, this is not something one figures out right away, especially
not as a Rust novice just starting to grasp the ownership rules. The good thing
is the compiler won't let faulty code compile, but it's still not very intuitive
and friendly. I myself needed several rounds of negotiation with the borrow
checker to find a solution even though I know the low-level principles what
hides behind an `async` block. There might be simpler solutions possible.

The proper solution here is probably to use [async
closures](https://github.com/rust-lang/rust/issues/62290), because then the code
would have only 2 levels, not 4 and become much more obvious.

```rust
let make_svc = make_service_fn(async move |_conn| {
    let handler = Arc::clone(&handler);

    Ok::<_, Infallible>(service_fn(async move |req| {
        let handler = Arc::clone(&handler);
        Ok::<_, Infallible>(router.route(req).await)
    }))
});
```

But this is not stabilized yet (and I'm not sure *if* it'll work). So until
then, we are probably left with documenting this pitfall in a well visible
place.

## Closing thoughts

I don't want to be negative ‒ it's been a long way from how the `hyper-0.12`
code looked like. It's almost pleasant to use. But it seems there are some more
rough edges to polish.

[hyper]: https://hyper.rs
[tokio]: https://tokio.rs
