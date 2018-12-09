# The Spirit tutorial

Don't worry, this is not about occult magic. As I'm developing the
[`spirit`][spirit-crate]
crate, I've decided a small tutorial would really help people to use it. So,
this post serves a dual purpose: to show how to use the library (and why) and to
ask for help with it.

## What does spirit do

I've already tried to introduce it in a [previous post][spirit-1]. The purpose
didn't change since then, though some details did.

In short, when writing a daemon or a service, we have the „muscle“ of the
application ‒ whatever we write the daemon for. And we have a whole lot of
infrastructure around that: logging, command line parsing, configuration. And
while there are Rust libraries for all that, one needs nontrivial amount of
boilerplate code to bridge all this together.

Spirit aims to be this bridge. It takes care of signal handling, of combining
multiple pieces of configuration together with command line overrides, it allows
for reloading the configuration at runtime. In short, it is the glue nobody
really wants to write every time. It doesn't do much itself, it just ties the
readily available libraries together. The aim is to save time with the boring
stuff and to provide the boring stuff with some bells and whistles one wouldn't
really care to write were it just for this one daemon.

## Status of the library and what you can do

The library is past the very early experimental state. I even dare to use it in
a production software. The high level design will probably stay the same or very
similar as it is, though the API itself might get some changes over time. It
feels like it helps a lot with the boilerplate in some cases.

On the other hand, big chunks of functionality are still missing and it feels
awkward to use at times. There are likely bugs, the documentation is
unsatisfactory and I'm just not good at writing useful log messages.

In other words, the library needs some users, experimentation and people willing
to help with polishing, fixing, smoothing rough edges, etc. Opening issues about
what doesn't work or what your use case is helps. Opening PRs to fix them, to
fill in TODOs in documentation or to add tests helps even more. I have some
ideas what needs to be done (some of them more concrete than others). Some of
them are hard, some of them are easy. If you want to help out but don't know
how, please contact me (through [github issue][repo] or over the [email],
but it might take me a day or two to answer sometimes), I'll be glad to discuss
what to do, how to do it or even help learning some Rust on the way if it's what
you need (or accept advice on how to do a nicer API if that's what you feel I
need 😇). Every bit counts and it's definitely more work than I can do alone in
my free time.

Also, I've only tried it on Linux. It *should* work on other Unix systems.
Changes to make it work on Windows might be needed.

## How to use it

Before we start writing code, few words about how the library ‒ or more exactly,
group of libraries ‒ works.

The core [`spirit`][spirit-crate] gives us a singleton [`Spirit`] object (well,
it doesn't *force* you to have just one, it simply doesn't make any sense to
create more) that manages the configuration of the application, its lifetime and
signals. The singleton is created by a [`Builder`] that allows to bootstrap it,
give it some basic information (like where to look for configuration files if
none are given on the command line or how does a default configuration look
like) and install bunch of callbacks for when configuration changes, when a
signal happens, when the application should terminate, etc. Then it runs a
provided application body with all these details taken care of.

The command line options and configuration are described by rust structures
implementing [`StructOpt`] and [`Deserialize`] respectively. They are type
parameters of both [`Builder`] and [`Spirit`] and either can be plugged by the
[`Empty`] structure if not needed.

So, how would this look like? Something like this:

```rust
use std::time::Duration;

use serde_derive::Deserialize;
use spirit::{Empty, Spirit};

fn default_interval() -> Duration {
    Duration::from_secs(1)
}

#[derive(Clone, Debug, Default, Deserialize)]
struct Cfg {
    message: String,
    #[serde(with = "serde_humanize_rs", default = "default_interval")]
    interval: Duration,
}

fn main() {
    Spirit::<Empty, Cfg>::new()
        .on_terminate(|| println!("Good bye"))
        .run(|spirit| {
            while !spirit.is_terminated() {
                let cfg = spirit.config();
                println!("{}", cfg.message);
                std::thread::sleep(cfg.interval);
            }
            Ok(())
        });
}
```

Then, if you create a configuration file with this content:

```toml
message = "Hello"
interval = "2s"
```

And run it:

```
cargo run -- cfg.toml
```

It'll keep saying `Hello` all over, until you press `CTRL+C`. Then it'll say
`Good bye` and exit. If you change the configuration file (while the program is
still running) and send a SIGHUP to it, it'll adapt to it ‒ it'll start saying
the new message if you changed that.

There are few things of note here:
* The [`serde_humanize_rs`] thing allows us to write durations in more human
  fashion, with units. Nothing related to [`spirit`][spirit-crate], but handy
  anyway.
* We run the [`spirit.config`][config] every iteration to get us an up to date
  config. This call is very cheap ‒ it is just a smart pointer into the version
  spirit holds for us. If we took one at the beginning, it would hold the
  *original* config from start of the application. The smart pointer returned
  from the `config` call doesn't change under our hands, therefore we can get a
  new one when we are ready for it, but it'll be in consistent state (eg. fully
  old or fully new).
* If you play with it, you'll discover that despite us not asking for any
  command line options, we already got *some*, mostly to provide `--help` and to
  specify configuration or configuration overrides.
* You can pass multiple configuration files. They are composed together, the
  latter ones overriding the earlier ones. So you can have a company-global
  configuration and machine-specific overrides, for example.
* With a little more configuration, we could also pass configuration
  *directories*. Every time the configuration is loaded, they are scanned for
  configuration files and these are loaded. You know, the style that you have
  eg. `/etc/cron.d` folder and cron loads all of them. You could ask spirit to
  extract configuration from the environment variables, to support deployment in
  docker cloud.
* If something goes wrong, the application will just *silently exit*. This is
  because [`Spirit`] uses [`log`] under the hood and we haven't set any logging
  up. We'll look into it soon.
* When we press `CTRL+C`, it doesn't exit right away, but after it waits the
  interval. We could do something about that, like having a [mpsc], doing
  [`recv_timeout`] in the loop and sending an „interrupt“ signal in the
  [`on_terminate`] hook. But let's leave it as an exercise for the reader.

### Active and passive configurations

Spirit by default assumes than *any* part of configuration can change at
runtime. For some daemons this is a must, because shutting them down is a big
hassle (postgress can take tens of minutes to start up again so you don't want
to restart it just to tweak the configuration a bit, apache lets you add new
virtual hosts without stopping serving the previous ones). Sometimes it is just
convenience ‒ if you have a service in production and it is misbehaving, it is
nice to be able to turn debug logging on in the relevant part while it is still
running.

If you don't like it, you can either turn the background signal processing
[thread off][build], or you can [selectively warn][immutable_cfg] the user about
pieces of configuration you're not able to adapt to.

If you do like it, you can adapt to new configuration in two ways. We've seen
the passive way above. Simply, the next time the configuration is needed, a new
value automagically appears in the configuration and we just use it. That's the
easy way, but not always enough.

The active way is registering a callback to be notified of the changes. There
are actually two, with different power. The simpler but less flexible one is
[`on_config`]. It is just told the configuration changed when it already
happened, like this:

```rust
.on_config(|cmd_line, new_cfg| {
    debug!("Current cmdline: {:?} and config {:?}", cmd_line, new_cfg);
})
```

The more complex one is [`config_validator`]. That one is run as part of loading
the configuration and it can *refuse* it. In such case, errors are printed and
the old configuration stays until the user fixes the problem and tries to load
it again. It also can tweak the new configuration before it is applied (like,
putting values within limits and warning about it instead of outright refusing
it) and it can schedule an action to happen once all the validators have run,
either on success or failure. The idea is, some configuration needs to be *tried
out* to see if it works, so the success allows to install it and failure to roll
it back in case some other validator said no.

### Helpers

A helper is something that modifies a builder. A `FnOnce(Builder) -> Builder` is
a helper, but more things can be so. This allows parts of configuration to plug
themselves into the builder without knowing each other. It also allows libraries
to provide pieces of functionality that can be plugged in with the [`with`]
method. That allows the library to register multiple callbacks at once, in
interdependent manner.

There are also configuration helpers. These are kind of building blocks for the
configuration, a fragment that can be put in there. Then, when registered with
an extractor function (one that extracts the fragment out of the whole
configuration), an action (what that is depends on the type of fragment) and a
name, they do the magic to handle reloading that bit of configuration.

An example can be a fragment to configure a TCP listening socket. The action
would be what happens with a new connection. All the rest ‒ binding to a port,
configuring a lot of details about how to listen and doing all the accepting,
shutting it down when it changes and building a new one, etc, is done by the
helper.

The other example is logging.

### Logging

While there are few helpers directly in [`spirit`], most of them live in other
related crates. So, let's go shopping ‒ we want [`spirit-log`] here to give us
logging.

So, first, we import it:

```rust
use spirit_log::{Cfg as Logging, Opts as LogOpts};
```

Then, we put the fragments into our configuration structure. We also want to put
the `LogOpts` into our command line options structure (we have to create one,
`Empty` is no longer good enough). We wouldn't have to, but it allows the user
to override logging on command line, which is convenient when trying things out.

```rust
#[derive(Clone, Debug, StructOpt)]
struct Opts {
    #[structopt(flatten)]
    log: LogOpts,
}

#[derive(Clone, Debug, Default, Deserialize)]
struct Cfg {
    message: String,
    #[serde(with = "serde_humanize_rs", default = "default_interval")]
    interval: Duration,

    #[serde(flatten)]
    log: Logging,
}
```

Cool, so now we can add this to our configuration, if we want to log both to
stderr and a file, with different options.

```toml
[[logging]]
level = "DEBUG"
type = "stderr"

[[logginng]]
level = "INFO"
type = "file"
filename = "/tmp/example.log"
clock = "UTC"
format = "machine"
```

But this'll only allow *parsing* the configuration. We need to make the
configuration active. First, let's write our extractors ‒ little functions that
take the whole configuration or command line structure and produce the relevant
bit. We could use an in-place closure, but the expected signature allows us to
actually write it as a method of the structure, which looks more tidy.

```rust
impl Opts {
    fn logging(&self) -> LogOpts {
        self.log.clone()
    }
}

impl Cfg {
    fn logging(&self) -> Logging {
        self.log.clone()
    }
}
```

And then just put it into the builder. That one will take care of initializing
our loggers and replacing them when the configuration changes. It'll even reopen
the log files on `SIGHUP`, which makes integration with [logrotate] seamless.

```rust
.config_helper(Cfg::logging, Opts::logging, "logging")
```

### Other helpers

Besides logging, there are already helpers for other things:

* Daemonization (going into background), in [`spirit-daemonize`].
* Some tokio integrations, mostly running the runtime and listening sockets, in
  [`spirit-tokio`]. Things like connection pools are planned.
* Hyper integration, which allows configuring a hyper server (or multiple ones),
  in [`spirit-hyper`].

And there's a long wish-list for other helpers and integrations. Not all of them
are obvious how to do or thought through. This is part of why it seems a lot of
work to make Spirit more usable ‒ it would be great if one could just come,
throw several configuration helpers/fragments in there and be done with
everything but the application specific logic.

* Something that can dump parsed configuration (and exit), eg. `--dump-config`.
* Something that can examine the configuration structure and show it to the user
  or generate some kind of annotated default configuration. A possible venue
  there is either creating additional trait to describe the documentation, or
  try to extract the structure from [`Deserialize`] through a fake
  deserialization format.
* Metrics. I'm experimenting with [`dipstick`], but different one is of course
  an option (or having multiple ones).
* Some applications might prefer [`slog`] over [`log`].
* [`reqwest`] ‒ keeping a global, configured HTTP client around.
* Some integration with [`sentry`]. However, reconfiguring that one at runtime
  poses challenges.
* Both [`spirit-tokio`] and [`spirit-hyper`] would benefit from something that
  can add TLS encryption, possibly as an intermediate layer.

And I'm sure people will discover more things that could be reusable and shared
between people.

### The full example

It's still longer than I'd have liked, but considering how much functionality it
provides already (eg. the logging configuration, composing of configuration,
etc), it's not that bad. Some derive might help with that (but someone would
have to write it first).

```rust
use std::time::Duration;

use log::debug;
use serde_derive::Deserialize;
use spirit::Spirit;
use spirit_log::{Cfg as Logging, Opts as LogOpts};
use structopt::StructOpt;

fn default_interval() -> Duration {
    Duration::from_secs(1)
}

#[derive(Clone, Debug, StructOpt)]
struct Opts {
    #[structopt(flatten)]
    log: LogOpts,
}

#[derive(Clone, Debug, Default, Deserialize)]
struct Cfg {
    message: String,
    #[serde(with = "serde_humanize_rs", default = "default_interval")]
    interval: Duration,

    #[serde(flatten)]
    log: Logging,
}

impl Opts {
    fn logging(&self) -> LogOpts {
        self.log.clone()
    }
}

impl Cfg {
    fn logging(&self) -> Logging {
        self.log.clone()
    }
}

fn main() {
    Spirit::<Opts, Cfg>::new()
        .on_terminate(|| println!("Good bye"))
        .config_helper(Cfg::logging, Opts::logging, "logging")
        .on_config(|cmd_line, new_cfg| {
            debug!("Current cmdline: {:?} and config {:?}", cmd_line, new_cfg);
        })
        .run(|spirit| {
            while !spirit.is_terminated() {
                let cfg = spirit.config();
                println!("{}", cfg.message);
                std::thread::sleep(cfg.interval);
            }
            Ok(())
        });
}
```

Also, there's a bit longer [example] in the repository, showing possibilities of
some more helpers.

[spirit-crate]: https://crates.io/crates/spirit
[spirit-1]: /2018/09/03/Announcing-Spirit.html
[repo]: https://github.com/vorner/spirit
[email]: mailto:vorner@vorner.cz
[`Spirit`]: https://docs.rs/spirit/~0.2/spirit/struct.Spirit.html
[`Builder`]: https://docs.rs/spirit/~0.2/spirit/struct.Builder.html
[`StructOpt`]: https://docs.rs/structopt/~0.2/structopt/trait.StructOpt.html
[`Deserialize`]: https://docs.rs/serde/~1/serde/trait.Deserialize.html
[`Empty`]: https://docs.rs/spirit/~0.2/spirit/struct.Empty.html
[`serde_humanize_rs`]: https://crates.io/crates/serde-humanize-rs
[config]: https://docs.rs/spirit/~0.2/spirit/struct.Spirit.html#method.config
[`log`]: https://crates.io/crates/log
[mpsc]: https://doc.rust-lang.org/std/sync/mpsc/index.html
[`recv_timeout`]: https://doc.rust-lang.org/std/sync/mpsc/struct.Receiver.html#method.recv_timeout
[`on_terminate`]: https://docs.rs/spirit/~0.2/spirit/struct.Builder.html#method.on_terminate
[build]: https://docs.rs/spirit/~0.2/spirit/struct.Builder.html#method.build
[immutable_cfg]: https://docs.rs/spirit/~0.2/spirit/helpers/fn.immutable_cfg.html
[`on_config`]: https://docs.rs/spirit/~0.2/spirit/struct.Builder.html#method.on_config
[`config_validator`]: https://docs.rs/spirit/~0.2/spirit/struct.Builder.html#method.config_validator
[`with`]: https://docs.rs/spirit/~0.2/spirit/struct.Builder.html#method.with
[`spirit-log`]: https://crates.io/crates/spirit-log
[logrotate]: https://linux.die.net/man/8/logrotate
[`dipstick`]: https://crates.io/crates/dipstick
[`spirit-daemonize`]: https://crates.io/crates/spirit-daemonize
[`spirit-tokio`]: https://crates.io/crates/spirit-tokio
[`spirit-hyper`]: https://crates.io/crates/spirit-hyper
[`reqwest`]: https://crates.io/crates/reqwest
[example]: https://github.com/vorner/spirit/tree/master/examples/hws-complete
