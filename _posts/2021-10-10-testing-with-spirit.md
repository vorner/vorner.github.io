# Testing with Spirit

There are many testing methodologies and even more testing frameworks. This is
not about one of them. This is rather about somewhat pragmatic
duct-tape-thermal-glue approach to testing services that use the [spirit]
libraries.

Testing is integral part of software development. Not only because it's a well
known best practice and ensures better quality of the result. It is often faster
to write few tests than repeatedly run the application manually.

With Rust, unit testing is particularly easy ‒ one doesn't need to go hunting
for a testing framework, integrate it in a build system, and all that. It's a
matter of putting a `#[test]` on a function.

But unit testing is not everything. It's the perfect thing for algorithms,
data structures and logic. But applications often contain other code than that ‒
dealing with OS, external services and external world.

These are harder to test. Not only is the situation more complex, every testing
framework tutorial shows how to test something simple like a red-black tree. We
want to talk over the network, delete files, check the application starts up
with given config.

Mocking might help a bit here, but it's hard in Rust and often doesn't actually
test the real situation. If I have a wrong assumption about behaviour of an
external system, I'd write the mock with that assumption too and then, when the
application is put into production and meets the real world, all kinds of hell
breaks loose.

On the other hand, one can have the infrastructure set in a way to allow testing
‒ either some kind of A/B testing with automatic rollback, testing environment
with mirror of the traffic, something like that. While this is really valuable
and often needed part of the picture, it's also too heavy-weight for fast
iteration on a problem. Deployment to testing environment and watching metrics
is something I want to do before hitting the big button to really go to
production, not something to be done 20 times a day.

So we want something that tests the whole or almost whole application ‒ starts
it up with given config, sends few requests, sees that these are successful (or
unsuccessful in the expected way) and contain the right response. And we want
this as part of the tests a developer can run on their machine without too much
fuss.

In some previous projects, we made a harness that compiled the whole
application, fed it a (customizable) test configuration, attached to the
program's log output (both for checking correctness and „synchronization“), run
the tests, checked the responses, logs, etc. It was usually written in some
„glue“ language (mostly Python). This worked reasonably well and writing tests
for that was not that much effort. But it still had several big disadvantages ‒
persuading the build system to inject the right dependencies (compile before
tests), parsing logs is cumbersome and fragile, it runs rather slowly. And
it was always a big pain to set up at first.

It works for complex multi-year projects and it may even me *the* difference
between success and the project falling apart. But it wouldn't work in a team
where there's a new service to write and launch every 2 months.

## Testing mode of Spirit

The [spirit] crate family is meant to help kick off services (often http based,
but not necessarily) fast, but with full featured lifetime and configuration
management. Few lines of additional code and one gets logging, metrics, layered
configuration ‒ all fully configurable and reloadable at runtime. The `Spirit`
works in somewhat singleton-ish way.

But it doesn't _have to_. As I was trying to find a way how to create few tests
for the whole or almost whole application fast and then move on my way to other
things, I've discovered that Spirit was quite well situated to do it (yes, as
the author of Spirit, I admit I haven't _designed_ this feature, I've
_discovered_ it after the fact). It can start and stop the application, it
manages the configuration. I only needed adding few small APIs to allow running
with provided configuration file (or configuration not in file), mock command
line arguments and not to install signal handlers.

Using it is much less work than setting up the abovementioned Python monster
system. Besides, it runs the „application“ inside the process like any other
usual unit and integration test, which has two advantages. It completely
sidesteps the problem with build system (having to build the application first
before running tests). It also makes the tests run significantly faster, both
because another process doesn't have to be started, waited for the right log
message to appear, and shut down, but also because the tests can easily run in
parallel (while parallel running of the „Python monster“ solution is, in theory,
also possible, we never found the time to actually implement it).

So let's have a look at how Spirit can be used to not only control the
application, but also test it.

## Prerequisites

Let's say you have a Spirit-based service or at least an understanding how one
is made. If you don't know it well enough, there's an [tutorial] in its
documentation and it should provide the basics.

## Preparing the service

First, let's refactor the service in a way that all the setup that needs to
happen both in tests and in the real application is extracted into a function.
Similarly, we'll extract the „body“ of the application. Something like this
(with just *a bit* more code and functionality):

```rust
type SpiritBuilder = spirit::Builder<MyConfig, MyCmdOpts>;
type Spirit = spirit::Spirit<MyConfig, MyCmdOpts>;

fn setup(builder: SpiritBuilder) -> Result<SpiritBuilder, AnyError> {
    builder
        // Registering the hooks, pipelines, all that stuff.
        .on_config(..)
        .on_terminate(..)
        .with(Pipeline::new("some-subsystem"). ..)
}

fn run(spirit: &Arc<Spirit>) -> Result<(), AnyError> {
    while !spirit.is_terminated() {
        // Some dummy application code
    }
}
```

Then, the real application looks something like this

```rust
spirit::utils::support_emergency_shutdown()
    .expect("Installing signals isn't supposed to fail");
Spirit::new()
    .config_all_exts()
    // Yes, the setup function conveniently has such a signature that it
    // satisfies the `Extension` trait.
    .with(setup)
    .run(run);
```

## Starting a testing instance

When testing, we want to do the following:

* Set a different configuration. As we didn't use the default configuration, we
  can abuse the `config_defaults` as the easiest way to inject it, but there are
  others. Some tips what to configure can be seen below.
* Override command line arguments. We don't want the ones the process (the test
  binary) received, we want to control them from the test.
* Build the `spirit` object without the background thread. The thread usually
  manages reloading of configuration and shutting down the application based on
  signals and we don't want these. We'll manage that on our own from the test.
* Run the application in another new thread so the test's thread is not blocked
  and we can do the testing in there.

It'll look something like this.

```rust
// A yaml config "file"
const TEST_CFG: &str = r#"
[section]
option = "hello"
"#;

// Using the type def Spirit = .. from above
let app = Spirit::new()
    // Using the defaults as the easiest way to inject things when we don't them
    // in the real app.
    .config_defaults(TEST_CFG)
    // Inject command line options (empty ones here).
    .preparsed_opts(Empty::default())
    // Application specific setup
    .with(setup)
    // False -> we don't have the background management.
    .build(false)
    .expect("Failed to create the test spirit app");

let spirit = Arc::clone(app.spirit());

// This is a RAII guard for the app running in the background thread.
let _running = app.run_test({
    let spirit = Arc::spirit(&spirit);
    move || {
        run(&spirit)
    }
});

// Here we do the actual testing

// And here the _running is dropped, which shuts down the managed application.
```

The [`run_test`] method isn't really that special, it starts the „application“
in a separate thread and returns a RAII guard object that calls [`terminate`] on
drop and waits for the thread to finish. It really just saves a bit of boiler
plate.

## Synchronization

If the test was written the way above, it would be flapping. The problem is, the
`run` runs in a different thread and the test could start sending requests
before the application it is ready. I usually use a channel to signal „I'm ready
to be tested“ from within there. When that moment is depends on the application.

* You may need to pass it inside the real `run` method, maybe as
  `Option<Sender<()>>` ‒ not used in the real application, but signaled in
  tests.
* With tokio based applications, the `run` function only sets up everything and
  the actual application runs inside a [tokio runtime][spirit-tokio] that's
  managed by spirit. The `run` function terminates fast and it is possible to
  signal being ready for tests after it terminates.

```rust
let app = Spirit::new()
    // Using the defaults as the easiest way to inject things when we don't them
    // in the real app.
    .config_defaults(TEST_CFG)
    // Inject command line options.
    .preparsed_opts(Empty::default())
    // Making sure we have a tokio runtime overriden with a single-threaded one.
    // Not important, but makes the test a bit more lightweight.
    .with_singleton(spirit_tokio::Tokio::SingleThreaded)
    // Application specific setup
    .with(setup)
    // False -> we don't have the background management.
    .build(false)
    .expect("Failed to create the test spirit app");

let spirit = Arc::clone(app.spirit());

// Channel from the standard library, not from tokio
let (sender, receiver) = mpsc::channel::<()>();

// This is a RAII guard for the app running in the background thread.
let _running = app.run_test({
    let spirit = Arc::spirit(&spirit);
    move || {
        // This just sets up few tasks in tokio to listen for requests.
        let result = run(&spirit);
        // Ignore the other end being gone.
        let _ = sender.send(());
        result
    }
});

// Wait for the app to be ready.
receiver.recv().expect("App died during setup");

// Here we do the actual testing
```

## Listening ports

By default, Rust runs tests in parallel. That means if each test starts an
"application" which listens on some port, they'll fight over the ports they
listen on. Even if we make sure not to run them in parallel, some other
application could use the port we picked for testing.

One approach would be to assign a different port to each test. That would make
the tests not collide with each other, but there would still be a risk of
something else in the system sitting on that port.

There's another trick. If we bind to port 0, the OS assigns some arbitrary
free port (0 is a special value). We could allocate the needed socket this way
and pass it into the application, but that would probably need a lot of changes
to the application.

Instead, we'll do:

* Allocate the socket with port 0.
* Find out which port got assigned.
* Close the socket.
* Put the port into configuration and let the application use it.
* Now we know the port to send our requests to.

This is a hack and it is not guaranteed to always work ‒ because we give up the
port for a while and in theory the OS could give it to someone else in between.
But in practice this works quite well, because the arbitrary ports are assigned
in some round-robin fashion.

```rust
// Ask the OS to pick a free port
let socket = TcpListener::bind("127.0.0.1:0".parse().unwrap()).unwrap();
// Find out which port it picked
let port = socket.local_addr().unwrap().port();
// Close the socket
drop(socket);
// So we can now re-use it in configuration
let config = format!(
    r#"
    [listen]
    port = {}
    host = "127.0.0.1"
    "#,
    port,
);
...
```

## Writing multiple tests

The above is not that bad, but it is still a bit verbose to repeat into multiple
tests. What I use is a wrapper function around the test that manages the
application, something like this:

```rust
fn with_server<F: FnOnce(u16)>(f: F) {
    let socket = TcpListener::bind("127.0.0.1:0".parse().unwrap()).unwrap();
    let port = socket.local_addr().unwrap().port();
    drop(socket);
    let config = format!(
        r#"
        [listen]
        port = {}
        host = "127.0.0.1"
        "#,
        port,
    );
    let app = Spirit::new()
        .config_defaults(config)
        .preparsed_opts(Empty::default())
        .with_singleton(spirit_tokio::Tokio::SingleThreaded)
        .with(setup)
        .build(false)
        .expect("Failed to create the test spirit app");

    let spirit = Arc::clone(app.spirit());

    let (sender, receiver) = mpsc::channel::<()>();

    let _running = app.run_test({
        let spirit = Arc::spirit(&spirit);
        move || {
            let result = run(&spirit);
            let _ = sender.send(());
            result
        }
    });

    receiver.recv().expect("App died during setup");

    f(port);
}
```

## Closing thoughts

The [spirit] crate helps not only with service startup boilerplate, but it also
can help writing some integration or end-to-end application tests. Yes, it is a
bit dirty approach, but it doesn't add much more work. Dirty tests are better
than no tests 😈.

## Side note about spirit & contributors

There's a note that [spirit] looks for contributors. I haven't abandoned the
crate, I still maintain it (and, when I really need some new feature, I find the
time to add it). But it is not thriving either and I don't have the time to do
everything that _could_ be done to improve it myself.

So, if you find it useful, but lack some feature or integration with some other
crate (for example if you would like it to configure some other crate), extra
hands would be useful.

Most of the `spirit-*` crates in the family were born by first using the `*` in
some actual application and then extracting it out to a separate crate and
polishing. Therefore, if you already use `spirit` and configure some kind of
„subsystem“ others could be interested in, extracting it and sharing it would be
nice 😇.

[spirit]: https://crates.io/crates/spirit
[tutorial]: https://docs.rs/spirit/0.4.*/spirit/guide/tutorial/index.html
[spirit-tokio]: https://docs.rs/spirit-tokio
[`run_test`]: https://docs.rs/spirit/0.4.*/spirit/app/struct.App.html#method.run_test
[`terminate`]: https://docs.rs/spirit/0.4.*/spirit/struct.Spirit.html#method.terminate

