---
title: "Cleanup support in Signal Hook"
---
# Cleanup support in Signal Hook

It seems one of the duties maintainers have is making sure their libraries are
known. A library nobody uses just dies of loneliness. One of the way is to write
blog posts about the libraries, so here we go. This one is about [Signal Hook]
and its new support for cleaning up the signals.

## A little about the library

Signals are an old Unix API. It's also pretty terrible. I guess it seemed a good
idea back then and maybe even *was* a good idea back then. But the landscape
has changed since.

As with all Unix APIs, this one gives you enough rope to hang yourself. In a
bazillion different very subtle ways which lead to absolutely irreproducible but
highly spectacular effects like a single-threaded applications deadlocking or
corrupt memory allocator internals. The worst thing about it is it looks fine
most of the time (except when you demo the application to the customer).

You can either take my word that the API is quite painful to work with or read
my [previous post] about signals that goes into much more details.

The library exists to handle some of the challenges of using signals correctly
and offers some common patterns ready to be used. If you stick to the usual
patterns, you don't even have to write unsafe code.

## New features since the last post

There are three improvements worth mentioning since then.

* The library got split into two parts. The back-end [signal-hook-registry]
  doing the low-level lifting, and [Signal Hook] itself. The idea is the
  registry will never release an API breaking change and therefore will be in
  good position to handle the global resource of signal handlers even if the
  high level API changes and an application pulls in multiple incompatible
  version of [Signal Hook]. As a side effect, it allows others to build
  different abstractions on top of the signal multiplexing at its core and still
  co-exist with [Signal Hook] nicely ([tokio] does, for example).
* A minimal [Windows] support was added, big thanks to Masaki Hara. Yes, Windows
  have signals too, even though they are a bit weird there. The support is
  incomplete yet, but it's a start.
* Support for the cleanup pattern was added. It is what this post is about.

## Using the library

The most flexible possibility is using the [register] function directly. That
one allows to run whatever code inside the signal handler. The function is
marked as `unsafe` for a good reason, as it opens the floodgates to many of the
mentioned traps.

Usually it is enough to only mark the signal in question happened and notify
some other thread about it. The library provides three ways to do that without
using any `unsafe` on the caller side:

* Setting a [flag]. Something else must actively check that flag from time to
  time to notice.
* Writing into a [file descriptor]. If used with a pipe or socket, the other
  end can wake up an event loop like [tokio] or a sleeping thread.
* Getting an [iterator] over received signals. This is similar to the [file
  descriptor] above, but with much friendlier Rust interface. It can also be
  turned into an async Stream if a feature flag is enabled. This one currently
  works with tokio-0.1, but support for [tokio-0.2] is waiting in a branch for
  release. I'll gladly accept PRs to support [async-std], but the last time I
  looked it was impossible to integrate custom file descriptors into it.

These patterns are not perfect. For example, I'd really like to have a support
for [crossbeam-channel], as that one can multiplex multiple channels together on
the receiver end ‒ one wouldn't need to use a separate signal handling thread
nor [tokio]. It [seems non-trivial][crossbeam-channel-support], though.

But they seem to be good enough for most cases I've met. If you want to see
examples, follow the above links, the documentation contains them.

## The problem

All of this, including the direct approach, has one downside. Imagine a bug in
the application which causes it to somehow lock up ‒ maybe it enters an infinite
loop or deadlocks.

The application doesn't respond, so the user presses `CTRL+C`. Usually the
application would use one of the above patterns, stop whatever it was doing and
proceeded to do a graceful shutdown. But because ti is running in circles in the
infinite loop, it doesn't notice it should shutdown. The application still
doesn't react in any meaningful way, so the user presses `CTRL+C` again, with
much the same effect. Eventually the user would either use `kill -9` or reboot
the computer, depending on their knowledge and experience.

The trick here is to reset the signal handler to its OS default after receiving
the first signal. The default for many signals, including `SIGTERM` (the one
coming from `CTRL+C`) is to terminate the program.

If we do that than the stuck application will terminate (in non-graceful manner)
on the second `CTRL+C`. But if it's healthy and responding like it should, it
would terminate correctly after the first one.

To support this trick, the [cleanup] module was added (it seems the link doesn't
work yet, docs.rs has some problem ‒ it'll hopefully catch up soon; until then
you can still build the documentation locally). It allows resetting signal
handlers to defaults, either manually or registering the cleanup into the
current signal handler. The latter can of course be combined with whatever else
signal handling is done by [Signal Hook].

There are still some rough edges around this. Most important one is that the
cleanup is a one-way irreversible change,
[at least for now][ticket-irreversible]. But a program should terminate only
once during its lifetime anyway (🤔 that's actually an interesting challenge,
finding a way to terminate the same application multiple times 😈).

I believe this is a place for another thank you, this time to Justin Karneges.
While it was me who wrote the code, I didn't notice it needed writing in the
first places and wouldn't do it without the [ticket][ticket-restore]. Feature
requests are no less important than pull requests.

## Closing thoughts

If you need to ask if you need this library, the chances are you don't, because
most programs are fine with not touching signals at all. But if you want to set
them, be sure to give this library a look. It'll save you a lot of reading
documentation of fine details and hunting down weird crashes happening once in a
blue moon.

And if you find you have an idea how to improve a library (not just this one),
share it. Maintainers like pull requests and they like good ideas. Not
everything gets accepted all the times, but the interest from other people is
valuable feedback and fuel driving both the project and the maintainer forward.

[Signal Hook]: https://crates.io/crates/signal-hook
[signal-hook-registry]: https://crates.io/crates/signal-hook-registry
[Windows]: https://github.com/vorner/signal-hook/pull/19
[register]: https://docs.rs/signal-hook/0.1.*/signal_hook/fn.register.html
[flag]: https://docs.rs/signal-hook/0.1.*/signal_hook/flag/index.html
[file descriptor]: https://docs.rs/signal-hook/0.1.*/signal_hook/pipe/index.html
[iterator]: https://docs.rs/signal-hook/0.1.*/signal_hook/iterator/struct.Signals.html
[tokio]: https://tokio.rs/
[tokio-0.2]: https://crates.io/crates/tokio/0.2.0-alpha.6
[crossbeam-channel]: https://crates.io/crates/crossbeam-channel
[crossbeam-channel-support]: https://github.com/crossbeam-rs/crossbeam/issues/240
[ticket-irreversible]: https://github.com/vorner/signal-hook/issues/30
[cleanup]: https://docs.rs/signal-hook/0.1.*/signal_hook/cleanup/index.html
[ticket-restore]: https://github.com/vorner/signal-hook/issues/23
[async-std]: https://crates.io/crates/async-std
[previous post]: {% post_url 2018-06-28-signal-hook %}
