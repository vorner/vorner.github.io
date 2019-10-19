# Migrate spirit to version 0.4

This post can serve as a step by step migration guide from [spirit] 0.3 to
spirit 0.4. If you already have an application using the crate, read on.

If you haven't heard about the [spirit] library yet, it is a library to help you
manage your configuration in an application and have it reloaded at runtime. It
allows you to have the changes applied automatically and also to manage lifetime
of the application. You can read more about it [here][reloading]. In that case,
a migration guide won't help you much, but I'm planning on having a tutorial how
to start with the library soon.

I've been working on the next version for a while. This brings several breaking
changes and here I want to describe both why they happened and how to adapt your
code to them.

## The overall theme

Mostly, the changes don't bring any new functionality. The common theme of these
changes is cutting down on dependencies of [spirit] a bit and removing as many
foreign types from other pre-1.0 crates from the public interface, to prevent
*having to* do breaking changes. Some other changes are piggy-backing with it,
like removing deprecated functions or moving few things around in the module
tree.

These are changes tho the *base* crate of spirit most of the time. There will be
some further changes to, for example, [spirit-tokio] once [tokio] releases a 0.2
version. That change will however have limited impact (and it'll be possible to
use [spirit-tokio] versions for either version of [tokio] in parallel). Anyway,
these changes still have to be written, which I'll be happy to accept a helping
hand with.

## Preliminary: updating the dependencies

The chances are, you have several spirit's crates (both the main one and
extensions) as your dependencies. As the version bump of the base crate requires
version bump of all the `spirit-*` extension crates as well, you'll need to
update all of them at once.

One of the spirit's dependencies that weren't possible to hide is [structopt].
Than one went from 0.2 to 0.3 and spirit 0.4 goes with the latter. So update
that one too if you have it as your own dependency (you likely do). The
interface of [structopt] is almost unchanged between the versions, so the
version bump is likely the only change you'll have to do about it. If not,
little tweaking of attributes should do the migration there.

Once you do that, you're going to get a wallpaper of compiler errors. While the
changes in spirit's API are simple ones, they are quite loud in terms of how
much code needs to be (trivially) changed.

## The error handling

At the level where [spirit] fits into an application, many things can fail and
most of the errors are just passing through it, possibly getting some more
context on them. Eventually they are likely to end up in a log and optionally
stop the application. Spirit uses multi-layered errors, sometimes called cause
or error chains. It goes the way of putting all levels of the error into the
log, to provide as much information to the user as possible.

Previously, the [failure] was used to handle the errors. The library is really
convenient for use. However, it comes with several drawbacks. It provides its
own [Error] type that is not directly compatible with the standard library
types. This means that mostly the whole application is then *forced* to use
[failure]. Considering the crate is in its 0.1 and is expecting breaking
changes, this is kind of problematic. Second, it unconditionally depends on
[backtrace]. That crate is sometimes really painful to cross-compile as I've
noticed on multiple occasions.

So, from now on, [spirit] uses [spirit::AnyError] in its interface. Unlike the
[Error] from [failure], this is only a type alias for a boxed trait object from
the standard library (`type AnyError = Box<dyn std::error::Error + Send +
Sync>`). Using this gives the maximum possible forward compatibility and
flexibility ‒ as equivalent type aliases are considered equal and it only uses
things from the standard library, any other error handling library can be used
by whatever applications and dependencies. You don't need to depend on [spirit]
to return compatible error type from your crate, just use the same trait object
or a your own concrete error.

An application has two options how to adapt. One is to keep using [failure] on
its side and its [compat] method. This is less work for migration, but it has a
significant downside. The compatibility error type is not transparent for the
causes chain, something that is heavily used through spirit. That means at the
point of conversion, some error causes will get lost and not shown to the user,
possibly hiding valuable information.

The other option is to replace [failure] with some other error handling
libraries. The combination of [err-derive] and [err-context] covers quite a
large part of [failure]'s API surface and it's what I used when migrating my
applications, but as mentioned above, any other replacement should work.

If going the way I did, you'd do these things:

* Replace all occurrences of [Error] with [AnyError][spirit::AnyError], or
  equivalent.
* Wherever there's a new leaf error type defined with `#[derive(Fail)]`, use
  `#[derive(Error)]` provided by [err-derive].
* Wherever you add another layer to an error with `.context` or `.with_context`,
  import `err_context::prelude::*` and all the error types get the methods with
  the same names.

## The logging fragment can be renamed

If you were using the [spirit-log] to configure logging destinations, you had
something like this in your configuration:

```rust
#[derive(Clone, Default, Debug, Deserialize)]
struct Cfg {
    #[serde(flatten)]
    logging: Logging,

    // Other stuff here
}
```

Unfortunately, no matter how you you named the field, it was always called
`logging` in your configuration and there was nothing that could be done about
it. The new version reflects the changes of the name of the field, but at the
cost of changing the needed attributes. This is how it should look like now:

```rust
#[derive(Clone, Default, Debug, Deserialize, Serialize)]
struct Cfg {
    #[serde(default, skip_serializing_if = "Logging::is_empty")]
    logging: Logging,

    // Other stuff here
}
```

Also, due to the changes, if your field is named differently than `logging`,
you'll need to rename it either in your configuration or in the struct.

## Prelude pruning

Because spirit uses extension traits heavily to do the setup, it contains a
`prelude` module to import all of them. Previously, the prelude also contained
several concrete types. This is generally considered a bad practice. Currently,
only the traits are exported from the prelude and these are exported
anonymously. Therefore, there's no chance of collision of the trait or type
names.

However, due to this change you may need to import some types or traits
explicitly, the most likely ones are `use spirit::{Empty, Pipeline, Spirit}`.
The compiler should provide hints where any other types or traits live.

## Feature flag tweaks

Some parts of functionality (and dependency graph) were opt-out previously. The
new version is more aggressive in using feature flags ‒ more parts of the
functionality are configurable by them and some features that were previously
opt-out are now opt-in.

The most significant ones are:
* `hjson`, `ini` which are now turned off by default. These add support for
  relevant configuration file formats. The `ini` format is considered not very
  expressive to be of good practical use. The `hjson` is seldom used format that
  however brings a lot of very old and unmaintained dependencies. If you need
  them, you can turn them on.
* `syslog` on the [spirit-log] crate, which is off by default now, enables the
  support for sending logs to syslog.

## The spirit's background thread is now auto-joined by default

Currently, the main application wrapper (`spirit.run(...)`) will wait for the
background thread to terminate. This can, under some occasions, cause a
deadlock (caused by improper use of your application resources).

You have either the possibility of hunting it down, or changing the behaviour
back with `autojoin_bg_thread(Autojoin::Abandon)`.

## Warning on unused parts of the configuration

If some parts of the configurations are unused, there'll be a warning in the
logs. If you have such options there on purpose, you can turn this off with
`warn_on_unused(false)`.

The detection is best-effort and doesn't work fully reliably. In some cases
unused parts of configuration can slip undetected.

## All the rest

There are some other changes too. Some of them are bugfixes, some of them are
formally breaking changes. But I hope these should produce no errors in common
use cases.

There are also few things that moved around. They should be easy to fix, as
`rustc` is very capable of pointing the new location out in the hint messages.

If there are any significant difficulties not described here, please let me
know.

[spirit]: https://crates.io/crates/spirit
[reloading]: https://vorner.github.io/2019/08/11/runtime-configuration-reloading.html
[spirit-tokio]: https://crates.io/crates/spirit-tokio
[structopt]: https://crates.io/crates/structopt
[failure]: https://crates.io/crates/failure
[Error]: https://docs.rs/failure/0.1.6/failure/struct.Error.html
[backtrace]: https://crates.io/crates/backtrace
[spirit::AnyError]: https://docs.rs/spirit/0.4.0/spirit/error/type.AnyError.html
[compat]: https://docs.rs/failure/0.1.6/failure/struct.Error.html#method.compat
[err-derive]: https://crates.io/crates/err-derive
[err-context]: https://crates.io/crates/err-context
