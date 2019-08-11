# Runtime configuration reloading

A lot of programs need to read some kind of configuration at startup. It doesn't
really matter what kind of programs these are, being it network services, games,
command line tools for processing vast amounts of data. Usually, they use some
library to read the configuration file, probably using one of the formats
supported by [serde]. If the configuration is more complex (it should support
multiple different formats, or can be composed from several files ‒ imagine
things like `program.d/*.yaml`), it's possible to reach out for [config].

But the challenge doesn't end here. Some programs ‒ certainly not all, but some
‒ are long running. For these, restarting them to change configuration isn't
something you'd want to do. Imagine a web server or database server. Such a
thing processes a lot of queries at all times and a restart kills all the
currently executing ones, which results in errors to end users or suboptimal
performance due to retries somewhere. While even these need to be restarted
sometimes ‒ because the machine needs to be moved to a different data center, or
because a new version brings critical fixes, it's nice not to have to restart
them too often. Alternatively, it may be matter of personal pride that your
product can do it. You don't want to restart the web server just because another
endpoint needs to be added, or a rewrite rule tweaked.

At different times, being able to tweak configuration at runtime may help in
debugging. Let's say the service is misbehaving, but restart will probably make
the misbehaviour go away for at least a while. Being able to increase a log
level in part of the application while the problem is already happening can give
valuable insight that would be hard to get in other ways. The alternative would
be to restart it and wait until the problem starts happening again, delaying the
solution and producing huge amounts of uninteresting log material.

## First steps to configuration reloading

Obviously, we need to be able to load the configuration from the file or files
(or somewhere else). But let's assume this one is already solved, because we
needed that even without reloading.

Then, we need some kind of trigger to reload the configuration. The obvious but
usually *wrong* way is to use something like [inotify] to watch for changes in
the configuration file. This is problematic solution, because many people keep
saving the configuration file in the middle of editing in potentially
inconsistent state. Then the program will load the inconsistent state
automatically.

So we need some kind of manual trigger. The unix daemon convention is to send a
[SIGHUP] signal to the process. For command line applications, this signal means
„your terminal went away, you probably want to terminate“. There's no terminal
to speak of for unix daemons, so it got reused. Anyway, on `SIGHUP`, a daemon
usually reloads all its configuration and reopens log files (this is for
integration with [logrotate]). I recommend the [signal-hook] crate for listening
to signals, because signals are [*incredibly* error prone][signal problems] to
use correctly and this crate shields from most of the problems. As we want to
do the configuration reloading in a separate thread anyway (to not disturb the
main application while doing IO), this is how it could look like:

```rust
use signal_hook::SIGHUP;
use signal_hook::iterator::Signals;

load_initial_config()?;
let signals = Signals::new(&[SIGHUP])?;
thread::spawn(move || {
    for signal in &signals {
        match signal {
            SIGHUP => {
                if Err(e) = try_config_reload() {
                    error!("Failed to reload configuration: {}", e);
                }
            },
            _ => unreachable!(),
        }
    }
});
```

Alternatively, the program can have some way to talk to it ‒ send some RPC
command that triggers the reload. I don't know what the convention for reloading
configuration of a service on Windows is, as there's no `SIGHUP` there.

And finally, we need to decide what happens if the configuration is somehow
invalid. The snippet above suggests that we want to just log the errors and keep
going, likely with the previous configuration state untouched. Handling syntax
errors in the files is easy ‒ the error just gets propagated and nothing
happened. However some things need to be validated ‒ like, some options are
valid only if some other options are set to certain values, or some values need
to be in specific range and referred files need to be readable. A lot of these
things can be checked before applying the configuration. So our code would look
something like this:

```rust
fn try_config_reload() -> Result<(), Error> {
    let f = BufReader::new(File::open("/path/to/config")?);
    let new_config = serde_yaml::from_reader(f)?;
    validate_config(&validate_config)?;

    unimplemented!("Let's talk about this part in the rest of the post");
}
```

*These snippets are certainly not production quality. I haven't tried compiling
them (for one, they are missing bunch of imports). Furthermore, there are
unsolved details ‒ what happens if the background thread panics? Should the
application terminate in such case? How do we shut down gracefully? The errors
should be somehow enriched by some kind of context to be more informative. We
may want to be able to produce multiple validation errors in one go. Etc, etc…*

## Applying configuration

There are generally three flavours of configuration options.

### Initial only

Certain parts of configurations can be applied only once. As argued above, we
don't *like* these kinds of configuration. But sometimes we just don't have the
luxury, some things just can't be changed. Or maybe it's not worth the effort
and code complexity. An example may be number of threads in both
[tokio][tokio-threadpool] and [rayon][rayon-threadpool] thread pools. They can
be started once with specific configuration and don't allow changing that. I
guess it would technically be possible to scaling up and down, but nobody wrote
the code yet. Migrating the work from one thread pool to a new instance also
doesn't seem viable.

However if we otherwise allow reloading the configuration, it is good manners to
at least warn about ignoring the changes. Instead of applying the change, we
would compare the initial configuration and the new one and log a message if
they don't match, so the operator doesn't spend long hours figuring out why the
new configuration doesn't work as expected.

```rust
impl InitOnlyComponent {
    fn init(cfg: Cfg) -> Self {
        Self {
            cfg,
            // The rest of the stuff…
        }
    }

    fn reload_cfg(&self, new: &Cfg) {
        if self.cfg != new {
            warn!(
                "Config not updated, requested = {:?}, used = {:?}",
                new,
                self.cfg,
            );
        }
    }
}
```

### Configuration picked up on every use

Some options can be used directly from the configuration, there's no action for
*changing* it. An example could be a list of users and passwords. Every time
someone tries to log in, the program consults the current list.

To support this kind of configuration, we need to be able to exchange it from
the configuration reloading thread under the hands of the worker threads. The
obvious solution is something like a [`Mutex`] or [`RwLock`].

This solution has few performance disadvantages. On one hand, a mutex is fast,
but will prevent even concurrent readers from accessing the structure at the
same time. Therefore, if we have a lot of worker threads and there's a lot of
accessing of the configuration, they'll step onto each other's toes and block
each other.

On the other hand, `RwLock` supports concurrent readers and writers are going to
be really rare (as reload happens only as a result of manual human action) so we
probably don't *care* about the blocking from them. However, RwLock is
significantly slower to lock than mutex and the performance gets worse with many
concurrent readers (due to the CPUs arguing over one cache line with the lock
counter).

So I use the [arc-swap] crate. It allows to store an [`Arc`] into global storage
which readers can borrow from it. The borrow is [lock-free][lock-free] (under
usual conditions even [wait-free]) and on par with locking a mutex, even if
multiple threads are borrowing the same thing (there's no shared cache line
being *modified*, which is fine performance-wise). This is great for our use
case ‒ readers are always fast, even concurrent ones, and once we store a new
one, the future borrows will just borrow the new version (and, of course, the
old one won't be destroyed as long as someone still has it borrowed).

There's another related decision to make. If our application has many
independent parts (which applications usually do), we don't want each part to
know about the whole configuration, even the parts not relevant to it (this
breaks several rules of good design). One option is to create one `ArcSwap` for
each part of the application and then, upon reloading the configuration, push
the new version of a sub-config into each of the part separately.

The other option is to use the [access] module of [arc-swap]. That way there's
just one global configuration which is published atomically as a one coherent
whole. But that configuration is masked from the parts and they see only their
part of the configuration.

So, let's extend our snippet a bit.

```rust
// Before we load configuration, it'll contain the default config values.
static CFG: once_cell::sync::Lazy<ArcSwap<Cfg>> = Lazy::new(Default::default);

fn try_config_reload() -> Result<(), Error> {
    let f = BufReader::new(File::open("/path/to/config")?);
    let new_config = serde_yaml::from_reader(f)?;
    validate_config(&validate_config)?;

    // TODO: We'll add something here as well, see the next section.
    CFG.store(Arc::new(new_config));
    Ok(())
}
```

And we'll pass the relevant part of configuration to an application component like
this:

```rust
fn start_app_component<C: Access<ComponentCfg>>(cfg: C) {
    unimplemented!("The application logic goes here");
}

start_app_part(CFG.map(|cfg: &Cfg| &cfg.component_cfg));
```

### Configuration that needs active change

Some configuration needs an action to change from the old value to the new one.
An example might be changing the file into which the application logs. The old
one needs to be closed while the new one needs to be opened. Well, because of
that logrotate thing, we want to reopen the log file even if it *didn't* change,
but that doesn't make our life harder, we can just pretend it did.

Great, this calls for some kind of callbacks from the configuration reloading
routine, that are all called on the new configuration.

But here comes the hard part. Some of these things can *fail*. For example we
might not have the rights to open the log file. Or a port we listen on might
already be taken. If that happens, we want to consider the configuration
invalid. But there is no way to know it upfront, during the validation (it's
really hard with the log file and probably impossible with the port and either
way it would contain a race condition ‒ what if someone took the port between
checking and actually opening it). But we want to keep the original
configuration intact if anything fails ‒ so if the last callback fails, we want
the previous ones to have no effect. Let's invent time travel…

This smells a little bit like transactions. Usually, this is done by splitting
the callbacks into two parts ‒ one that opens the log file or starts listening
on the new port. The second one actually replaces the open log file in the
application. If anything fails, the prepared results of the previous callbacks
are just thrown away. The second (installation) phase is not allowed to fail.

In this scenario, we close the original file *after* we created and started to
use the new one. Some care needs to be taken about certain resources ‒ if
we *don't* change the port, our attempt to open it again (because we reloaded
the configuration for some other reason) would fail because it is already taken
*by us*. Therefore the callback needs to somehow recognize if any action needs
to be taken at all, probably by comparing the old and new configuration. This is
in contrast to the logging file, where we want to pretend it changes every time.

Let's extend our snippet even further then.

```rust
type Install = Box<FnOnce()>;
type Callback = Box<FnMut(&Cfg) -> Result<Install, Error>>;

fn try_config_reload<C>(callbacks: C) -> Result<(), Error>
where
    C: IntoIterator<Item = &mut Callback>,
{
    let f = BufReader::new(File::open("/path/to/config")?);
    let new_config = serde_yaml::from_reader(f)?;
    validate_config(&validate_config)?;

    let install_actions = callbacks
        .into_iter()
        // Call each config with the new configuration
        .map(|cback| cback(&new_config))
        // The collect to Result will stop at the first error and will return
        // the error. We short-circuit on that.
        // If no error is encountered, we get a vector of install actions.
        .collect::<Result<Vec<_>>, _>()?;

    // Activate everything
    for action in install_actions {
        action();
    }

    CFG.store(Arc::new(new_config));
    Ok(())
}
```

## That's a lot of code

Certainly, it is (notice that we didn't include the callbacks themselves). It is
not exactly an easy problem. I've noticed it some time ago and didn't want to
write it again and again.

This is also the reason why I started the [spirit] framework some time ago. It
actually does something more. It also manages the application lifetime (eg.
handles callbacks for termination and listens to SIGTERM and such).

Furthermore, it also provides fragments of configuration and implementation of
the callbacks for various parts of usual application ‒ things like logging,
being able to go to background as a daemon, listening for network sockets and
listening on HTTP using hyper. It's quite flexible in many ways (while there's a
recommended way how to use it, unlike many other frameworks this way is not
enforced and the application can be structured differently or opt-in only to
parts of the functionality).

Furthermore, it is able to generate help for both command line and configuration
options from the documentation comments and attributes on the structures into
which they are deserialized.

However, while using it saves me a lot of time when starting a new project,
there's still a lot to be desired and I can't say I'm happy about the library's
state. Mostly, I see these downsides:

* It brings in quite a lot of dependencies. Most of the time it would be the
  dependencies that the application would use anyway ‒ like the [spirit-tokio]
  crate (the crate containing the callbacks and configuration fragments for
  listening sockets ‒ the framework is split into parts based on what each one
  configures) depends on [tokio] and it makes sense that one would want to
  configure [tokio] only in case it is going to be actually *used*. But sometimes
  it brings in a bit more than absolutely necessary ‒ the [spirit-log] brings in
  support for logging to syslog, for example, and even if one doesn't need that,
  the dependency is still there.
* The API seems a bit awkward and complex at times. I guess this mostly needs
  iterating over it and improving over time.
* The selection of usual components is still very limited ‒ it supports [log]
  but not [slog] yet. I want to add support for the new async ecosystem too, not
  just the older version of [tokio] 0.1. Supporting [sentry] and [rustracing]
  and probably many more I'm not aware of right now would very much make sense.
* The documentation is hard to understand. I think this kind of thing mostly
  needs an up to date tutorial.

All in all, there seems to be a lot to do and I certainly don't have enough time
to do everything. On the other hand, I don't want to let it rot completely,
because it *does* save a lot of development time.

So if you have ideas how to *simplify* it so there's less of work or it is
easier to understand and document, I'd like to hear it. If you want to help out,
I'm all for it :-). There are not many specific tasks on
[github][spirit-github], but I'm willing to discuss what needs doing and how or
even help out understanding some arcane parts of Rust on the way (there
certainly are a few).

Or, if there are viable alternatives, I could let spirit retire. But so far I
haven't seen anything remotely matching my needs.

[serde]: https://serde.rs
[config]: https://crates.io/crates/config
[inotify]: https://en.wikipedia.org/wiki/Inotify
[SIGHUP]: https://en.wikipedia.org/wiki/SIGHUP
[logrotate]: https://linux.die.net/man/8/logrotate
[signal-hook]: https://crates.io/crates/signal-hook
[signal problems]: https://vorner.github.io/2018/06/28/signal-hook.html
[tokio-threadpool]: https://docs.rs/tokio-threadpool/0.1.*/tokio_threadpool/struct.Builder.html#method.pool_size
[rayon-threadpool]: https://docs.rs/rayon/1.*/rayon/struct.ThreadPoolBuilder.html#method.build_global
[`Mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[`RwLock`]: https://doc.rust-lang.org/std/sync/struct.RwLock.html
[`Arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html
[arc-swap]: https://crates.io/crates/arc-swap
[lock-free]: https://en.wikipedia.org/wiki/Non-blocking_algorithm#Lock-freedom
[wait-free]: https://en.wikipedia.org/wiki/Non-blocking_algorithm#Wait-freedom
[access]: https://docs.rs/arc-swap/0.4.*/arc_swap/access/index.html
[spirit]: https://crates.io/crates/spirit
[spirit-tokio]: https://crates.io/crates/spirit-tokio
[spirit-log]: https://crates.io/crates/spirit-log
[log]: https://crates.io/crates/log
[slog]: https://crates.io/crates/slog
[tokio]: https://crates.io/crates/tokio
[sentry]: https://crates.io/crates/sentry-rs
[rustracing]: https://crates.io/crates/rustracing
[spirit-github]: https://github.com/vorner/spirit
