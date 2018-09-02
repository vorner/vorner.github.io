# Announcing Spirit

Spirit is a crate that cuts down on boilerplate when creating unix daemons, with
support for live configuration reloading.

## A case study

To explain the motivation and what the crate does, let's do a small case study.
We want to create a Hello World Service ‒ the client connects over TCP and is
greeted. Let ~~steal~~ borrow and modify an example from tokio (you know,
because we want to scale to ridiculous number of parallel clients and such and
because tokio is cool).

```rust
extern crate env_logger;
#[macro_use]
extern crate log;
extern crate tokio;

use tokio::prelude::*;
use tokio::net::TcpListener;

fn main() {
    env_logger::init();

    // Bind the server's socket.
    let addr = "127.0.0.1:12345".parse().unwrap();
    let listener = TcpListener::bind(&addr)
        .expect("unable to bind TCP listener");

    // Pull out a stream of sockets for incoming connections
    let server = listener.incoming()
        .map_err(|e| eprintln!("accept failed = {:?}", e))
        .for_each(|sock| {
            let addr = conn
                .peer_addr()
                .map(|addr| addr.to_string())
                .unwrap_or_else(|_| "<unknown>".to_owned());
            debug!("Handling connection {}", addr);
            let written = tokio::io::write(sock, "Hello world\n")
                .map(|_| ())
                .or_else(move |e| {
                    warn!("Failed to write message to {}: {}", addr, e);
                    future::ok(())
                });
            tokio::spawn(written)
        });

    // Start the Tokio runtime
    tokio::run(server);
}
```

This is all-right as a prototype goes, but there's a whole bunch of work to be
done before this can go to production:

* We would like to have a real daemon, that can go into background (or not, as
  the user wants).
* A lot of things should be configurable. Like the message we want to send to
  the user, or the port we listen on.
* It would be good if it supported layered configuration ‒ like a common
  configuration for all the servers in one file and another file with local
  overrides. Well, maybe not in this very trivial example, but in general…
* It is convenient for debugging purposes (even operational debugging purposes)
  if parts of configuration can be overwritten with a command line switch.
* Oftentimes, secret parts (like passwords to other backends) of configuration
  aren't written into the file but passed to the service in an environment
  variable by some kind of cloud manager.
* The logging is good enough for testing, but we want to be able to send the
  logging to a file, or actually to multiple destinations and configure the log
  levels per destination.
* The code above doesn't support any kind of graceful shutdown ‒ it'll just kill
  all the connections, maybe in the middle of the message.
* A well behaved daemon is able to actually reload its configuration at runtime,
  when notified by SIGHUP.
* Log files should be reopened on SIGHUP, to integrate well with logrotate.
* All of the above requires some error handling, not just unwraps.

Did I forget about something? Probably. But it is an impressive laundry list
anyway. And it has nothing to do with the specific purpose of the service, this
is a laundry list we would have no matter what service we would be writing.

And sure, there are crates for most of these things around. But integrating them
into the program still requires some work and some plumbing code. We should be
spending our time writing the unique code, not some plumbing boilerplate again
and again.

The plumbing code is inside the `spirit` crate. It gets done most of the above
list, with only some configuration. You specify a structure where you'd like
your configuration to be loaded, a structure where parsed command line arguments
should be stored (if you don't want some of these, use the provided `Empty`
structure). Then, you can either hook come callbacks in or let some helpers
(fragments of configuration with code provided by `spirit` and some companion
crates) do all the reconfiguration stuff.

This is how the example looks like with `spirit`

```rust
extern crate failure;
#[macro_use]
extern crate log;
#[macro_use]
extern crate serde_derive;
extern crate spirit;
extern crate spirit_tokio;
extern crate tokio;

use std::collections::HashSet;

use failure::Error;
use spirit::{Empty, Spirit, SpiritInner};
use spirit_tokio::TcpListen;
use tokio::net::TcpStream;
use tokio::prelude::*;

#[derive(Default, Deserialize)]
struct Ui {
    msg: String,
}

#[derive(Default, Deserialize)]
struct Config {
    /// On which ports (and interfaces) to listen.
    listen: HashSet<TcpListen>,
    /// The UI (there's only the message to send).
    ui: Ui,
}

impl Config {
    /// A function to extract the tcp ports configuration.
    fn listen(&self) -> HashSet<TcpListen> {
        self.listen.clone()
    }
}

/// Handle one connection, the tokio way.
fn handle_connection(
    spirit: &SpiritInner<Empty, Config>,
    conn: TcpStream,
    _: &Empty,
) -> impl Future<Item = (), Error = Error> {
    let addr = conn
        .peer_addr()
        .map(|addr| addr.to_string())
        .unwrap_or_else(|_| "<unknown>".to_owned());
    debug!("Handling connection {}", addr);
    let mut msg = spirit.config().ui.msg.clone().into_bytes();
    msg.push(b'\n');
    tokio::io::write_all(conn, msg)
        .map(|_| ()) // Throw away the connection and close it
        .or_else(move |e| {
            warn!("Failed to write message to {}: {}", addr, e);
            future::ok(())
        })
}

pub fn main() {
    Spirit::<_, Empty, _>::new(Config::default())
        .config_ext("toml")
        .config_helper(Config::listen, handle_connection, "listen")
        .run(|_| Ok(()));
}
```

When you look at it, there are three sections (not counting the imports at the
top). The first part describes the structure of configuration, in form of
`serde`-deserializable structures. The `spirit` will take care of loading it and
updating it on `SIGHUP`. Notice the `TcpListen` part, which comes from the
`spirit-tokio` helper crate ‒ that one knows how to parse configuration for and
create a TCP listener (and change the ports at runtime, to reflect the
configuration).

The second part is handling of one TCP connection. It's almost the same as the
closure in the `for_each` in the above example, with few little differences.
First, it gets some more parameters in addition to the connection. An instance
of the `Spirit` singleton, which is used to read the up to date value of the
message to send. Then there's the one unused parameter of `Empty` type ‒ the
`TcpListen` configuration fragment allows to plug additional
application-specific configuration (as a type parameter) of each listening
socket and it is passed as this parameter.

And then there's the bootstrapping section, that creates the `spirit` object and
fires it off. To explain what happens there:

* `config_ext` is used when a configuration directory is passed to the program
  (on command line) ‒ all files with this extension are loaded and merged into
  configuration.
* `config_helper` plugs a helper into it. Here we provide a function that
  extracts the listening definitions (which already knows how to keep up to
  date), what to do with one connection and a name that'll appear in logs.
* `run` runs the application. The body here is empty, everything is handled by
  tokio behind the scenes for us (and that one is fed into spirit by the helper
  from previous line).

## Playing around with the example

The above example is also in the [git
repository](https://github.com/vorner/spirit/blob/master/spirit-tokio/examples/hws.rs)
‒ with few additions, like embedded default configuration. You can try it out.

So, let's run it without any configuration to start with, but with a
configuration directory set to your home directory ‒ so we can add configuration
later on:

```
cd spirit/spirit-tokio
cargo run --example hws -- -l debug -L hws=trace -L spirit=trace "$HOME"
```

What we did here is turning on debug to error output on debug level, but the two
interesting crates (the `hws` itself ‒ hello world service, and spirit) run on
trace.

You can connect to it on port 2345 and you should see the `hello world` message.

Now, add a configuration file (`hws.toml`) to your home directory:

```toml
[[listen]]
port = 7891

[ui]
message = "Bye bye"
```

And send SIGHUP to the service:

```sh
killall -s SIGHUP hws
```

You'll notice several log messages to scroll by. You are no longer able to
connect on the port 2345, but on the new one 7891. And you get a new message.

## The state of the crate and plans

I follow the „release often, release early“ rule. The crate is in early stages
of development ‒ so you'll find TODO notes scattered through the code and
documentation, many pieces of functionality are still missing. But I hope
this'll improve over time.

Also, the only available configuration fragments ‒ the magical helpers ‒ are for
TCP and UDP sockets. This should improve. Next versions (or, other crates,
actually) should provide more helpers ‒ for TLS sockets, `hyper`, `tokio-web`
and probably others.

Currently, only unix systems are supported. I don't know Windows and how the
services there work, but hopefully someone will implement it. If not, I'll at
least make sure it can compile and provide limited functionality.

## How you can help

First, by trying it out. I want to know if there's something to improve (well,
there likely is). [Report](https://github.com/vorner/spirit/issues) problems and
ideas how to improve.

If you like to write code, the repository contains some issues that can be
worked on (and if you want to help with them, but feel stuck, I'm offering
help).

And of course, adding new helpers (either into the repository by a pull request,
or as completely independent crates) helps too.
