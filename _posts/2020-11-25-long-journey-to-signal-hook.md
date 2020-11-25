# The long journey to signal hook 0.2

More than two years ago, I've [started to work][signal-hook-post] on the
[signal-hook] library. The motivation was, and still mostly is, signal handling
is hard to get right. I've had the privilege to learn enough about it to be able
to do it right, but I still don't want to do that on a regular basis every time
I start a new project (it is a lot of effort and lot of places to get caught in
a trap). So the library is an attempt to save both my own effort and to make it
possible for others less interested in these arcane ancient APIs to get signal
handling that actually works.

This spring I've thought it was about time the library goes 1.0. So I've
started, in the limited amount of free time I have, to work on some cleanups and
preparations. But during these a lot of questions and issues popped up, leading
to some changes in the API. As there was quite a few of them, I've not released
1.0, but 0.2 and I want to see if it works well for people before I commit to
stabilizing it for some long time.

That being said, while the changes are strictly speaking breaking, the chances
are that if you used the library previously, you would need to only bump the
version number of the dependency and it would work or only small cosmetic
changes would be needed (like, adding a type parameter here or there, importing
another crate instead but keeping the code, …).

## Testing wanted

If you use the library or have a good way to test it, I'd ask you to test it.
I'm the author and maintainer, which means I'm chronically blind to faults
in it 😇. If you find rough places in the documentation, the API, it doesn't
work for you, etc, please open an issue in [the repository].

Pull requests are welcome too, but I'd suggest you announce your intentions for
the changes before you start working on them. I've noticed that either the
signal handling is more complex than most people suspect at first or that I have
very high standards about the code going in. Either way, many pull requests
submitted without prior design end up in a discussion after the fact and many
substantial changes being done.

That being said, if you have a use case not covered by the library and it is
the scope of what it wants to be, I'll be more than happy to help you find the
best way how it would fit in and how to implement it without running into too
many potholes in the POSIX roads.

Speaking about contributions. There's somewhat minimal and unsatisfactory
support for the Windows OS. I don't have that OS, I know mostly nothing about
it, so I can't reasonably improve it, but it would be great if the library also
provided some kind of abstraction/papering over the OS differences. If you are
versed in these OS details and have some time to spare, I'd be very happy if you
could cover that area.

## What's new

So there are some changes since the 0.1 version that might be interesting.

The first one is, the library needs rustc 1.31 if compiled without the
`iterator` support and 1.36 with it. As the `iterator` module is probably the
API most people would want to use, the feature is on by default. As the rust
version goes, the „backend“ part of the library, [signal-hook-registry], still
compiles with 1.26.0 and has only [libc] as its dependency (the dependency on
[arc-swap] got dropped).

Then there are some minor cleanups too. But there are also two new big features.

## Extraction of the async support

When the `signal-hook` version 0.1 was released, there was only [tokio] 0.1.
So the library provided some integration to that runtime and gave one an
asynchronous stream over the incoming signals (even though tokio itself now
provides its own one).

Today there's a myriad of different runtimes with different versions, some of
them are not yet stable so putting support for them into a crate that wants to
stabilize and go 1.0 is problematic.

For these reasons, the current version doesn't *directly* contain support for
any of these runtimes. Instead it exposes enough of the [internal details] to
make it possible to bolt the support for whatever runtime on top of it from
another crate.

The same repository actually contains three such integration crates, and is
generally open to have more.

* [signal-hook-tokio]
* [signal-hook-mio]
* [signal-hook-async-std]

They have their own versioning and their own life cycles, therefore don't block
the main crate from stabilizing while eg. [tokio] is still not stable. After
some discussions and hard choices, it was decided that if the runtime is stable,
the relevant crate would provide support for only that newest version. But for
the cases where the runtime is still unstable, the same crate would support
potentially multiple versions in parallel. Each supported version needs to be
turned on by a feature. So if you want to integrate with `tokio-0.3`, you'd add
this to your `Cargo.toml`:

```toml
signal-hook-tokio = { version = "0.1", features = "support-v0_3" }
```

Here the thanks belong to [Thomas Himmelstoss], as the one who made the
transition to these segmented crates happen and added support for the additional
runtimes, in a series of really epic pull requests.

## Future support for additional information

Currently, the library provides information on the level of „signal number 6
happened, at least once“. But the OS provides, through the [`sigaction`] and
`siginfo_t`, a bit more information about the signal and the circumstances, like
what other process sent the signal, at what address did an illegal instruction
happen, etc. It is natural asking if the library could expose these in some way
(this was brought to the attention by a pull request trying out some other way
than the one described here).

While it is supported on the very low level, through [`register_sigaction`],
this doesn't really fulfill the role of making the API comfortable and easy to
use correctly.

So the `0.2` version is going to explore a way how to exfiltrate a bit more
information from the signal handler to the user.

You'll notice that the [`Signals`] iterator is now just a type alias. The
[`SignalsInfo`] thing that backs it up has an additional type parameter,
something called `Exfiltrator`. It is responsible for describing what
information and how is smuggled out of the signal handler (given all the
stupid constraints on what one *can't* do inside, the fact that not all the
information is available for all the signals and accessing the non-existing one
is *likely* UB, …).

A similar pattern is present at all the async crates, so by implementing an
`Exfiltrator`, one would get support for that kind of information not only for
the blocking iterator interface, but for *all* the asynchronous ones too
(including ones not maintained by me, potentially).

For now, this is not very useful. The crate contains only the single
implementation, providing the signal number. Furthermore, the trait is sealed
and all the methods hidden behind the seal, so it is not possible for users to
implement their own. The plan is to implement few more, providing some common
information, in releases soon to come. That will be used to validate the draft
of the trait and refine it. After that happens and there's enough confidence
that it can actually fly, the trait will get unsealed for users to implement.

What I envision, users would then use it in way somewhat like this:

```rust
let signals = SignalsInfo::<WithSender>::new(&[SIGTERM, SIGHUP])?;
for SignalWithSender { signum, sender } in signals {
    println!("Received {} from PID {}", signum, sender);
}
```

If you have a good use case for information that needs extracting from the
`siginfo_t` structure, you can bring it in, so the theory of how this'll be a
great interface can be tested.

## Conclusion

My plan to go 1.0 this year clearly didn't work and even the 0.2 took quite a
bit of time. But I still feel the 0.2 brings some goodies and is on the right
way towards eventual 1.0. However, if you want to bring the 1.0 closer, you can
definitely help by testing, reporting bugs, fixing documentation and similar
activities. There's a chance the 1.0 would come early next year.

[signal-hook]: https://docs.rs/signal-hook
[the repository]: https://github.com/vorner/signal-hook
[signal-hook-registry]: https://docs.rs/signal-hook-registry
[libc]: https://docs.rs/libc
[arc-swap]: https://docs.rs/arc-swap
[tokio]: https://docs.rs/tokio
[internal details]: https://docs.rs/signal-hook/0.2.1/signal_hook/iterator/backend/index.html
[signal-hook-tokio]: https://docs.rs/signal-hook-tokio
[signal-hook-mio]: https://docs.rs/signal-hook-mio
[signal-hook-async-std]: https://docs.rs/signal-hook-async-std
[Thomas Himmelstoss]: https://github.com/tfkhim
[`sigaction`]: https://www.man7.org/linux/man-pages/man2/sigaction.2.html
[`register_sigaction`]: https://docs.rs/signal-hook-registry/1.2.2/signal_hook_registry/fn.register_sigaction.html
[`Signals`]: https://docs.rs/signal-hook/0.2.1/signal_hook/iterator/type.Signals.html
[`SignalsInfo`]: https://docs.rs/signal-hook/0.2.1/signal_hook/iterator/struct.SignalsInfo.html
[signal-hook-post]: {% post_url 2018-06-28-signal-hook %}
[spring-cleanup]: {% post_url 2020-03-01-spring-cleanup %}
