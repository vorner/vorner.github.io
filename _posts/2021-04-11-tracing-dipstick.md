# Extracting tracing metrics to dipstick

While I enjoy programming, there are certain aspects that are less fulfilling
than others. When writing a „business“ service, there's quite a lot of
distracting boilerplate besides the actual _useful_ code. I've tried to cut
down on that previously with the [spirit] family of libraries, because I didn't
want to handle reading configuration, logging setup, etc every time a new
service was started.

But this one is not about [spirit]. Some time ago, I've noticed the appearance
of the [tracing] crate and was wondering since how much traditional
functionality could be derived from that single instrumentation of a program.
Logging, certainly. But I was more annoyed by the need to gather metrics through
the program. And I was specifically annoyed by the fact that I've had a log
message near each metric collection point ‒ if I want to count something, it's
probably important enough for it to appear in logs. So, shouldn't it be possible
to write that just once?

## Annotating the tracing statements

The [tracing] crate offers similar macros as the [log] crate, like `info!`. It
also adds macros to create so called spans ‒ the „normal“ macros are one-off ‒
they just happen (they are called _events_ in the tracing terminology). But
spans have a start and an end. They also nest into each other (well, they can do
some more crazy things for advanced tracking of what's happening in a program
and why, but that's not needed here).

Events can be used quite naturally to collect counters, they can contain values
for gauges, etc.

The spans are ideal for timing things, for nesting metric naming scopes and for
adjusting levels (eg. how many requests are being handled right now in
parallel).

The only problem is, how does the _bridge_ (let's call the thing that takes
[tracing] information and turns that into traditional metrics that can be put
somewhere into graphana a _bridge_) know which events and spans to consume?

What I've come up with is placing an attribute onto the span or event macro
call. The bridge can look at the attribute and decide what to do about it. It
has a small downside, though ‒ these are more like instructions for the bridge
than human-useful information directly, but they still end up in logs or other
places these things travel to.

```rust
let _span = info_span!("Handling yak", metrics.timer="yak").entered();
info!(metrics.counter="shave", "Shaving yak")
```

These would provide the timing information of the whole handling of each yak and
count how many yaks the program shaved.

I _think_ this can be used even in combination with the [instrument] procedural
macro to lower the amount of typing even more (but I didn't have
opportunity/time to play with that one yet).

## The dipstick bridge

Eventually, I've decided to roll up my sleeves and actually try writing a
bridge. I've used the [dipstick] metrics library previously, so I'm using that
one.

The crate is published as [tracing-dipstick], with an [example] how to set it up
and how to use it (with the list of [supported attributes]). It's quite early,
so there's a chance there could be some bugs. There are places where I believe
the overhead could be lowered. Some features might be missing (for example, one
might want to also measure the time an asynchronous task spends doing something
vs. waiting).

There's also one problem with filtering. The crate sits on top of
[tracing-subscriber], which is the thing that allows composing multiple tracing
"backends" together. It also allows for filtering of the tracing information by
severity. The issue is, it does so globally ‒ for all the backends. While
logging backend wants them to be filtered, this bridge likely wants to consume
_all_ the events and spans ‒ even if one doesn't want to _log_ each request the
service handles, it wants to _count_ them.

This may be solved in a future version of [tracing-subscriber] or maybe worked
around in a future version of the bridge. For now, it is possible to combine
logging and metrics by using the `logging-full` feature of [tracing] ‒ so
logging _doesn't_ go through the subscriber and is routed directly to the
traditional [log] crate.

Anyway, that looks as a good enough first step for trying the experiment out and
figuring if it makes sense to extend or not.

## Compatibility with other bridges

I've used the [dipstick] mostly because I've already worked with it before (and
because [spirit] knows how to configure it).

But the names of the attributes are quite general. It is conceivable that a
different bridge for different metrics backend will be created (by me or others,
as the need arises). In such case a whole application could be migrated to a
different backend by replacing just the initialization part, not by going
through all the instrumentation of the whole code. I guess it's another instance
of „There's no problem that couldn't be solved by another layer of abstraction“
thing.

## Desirable improvements

I've identified few areas that are annoying at best.

The first one is the problem with filtering, described above. That one is
already known and [tracing-subscriber] really wants to have an answer to that,
long-term.

The other one is, it would be nice to somehow _hide_ the `metrics.*` attributes
from the logs. But it's currently not possible to filter individual attributes,
only whole events and spans, as they travel through the subscribers. I'm not
sure if such feature is generally desirable, though, or if there's a way to
implement it in an effective way.

The last thing that comes to my mind is ‒ if one makes a typo in the attribute
name and writes something like `metrics.coutn` or `metric.count`, there won't be
a compilation error. The attribute (and therefore the event/span) will simply be
ignored by the bridge and nothing happens. It could be solved in part by
checking the attributes for `metrics.*` and reporting the ones that are not
understood at startup, but that's still not great.

## The future

So, what happens now? I want to experiment with it a bit more and see if it is
usable. I have a good first feeling from it, because it cuts down on the part of
programming I don't like ‒ all that instrumentation boilerplate thing.

It's another crate that I have to maintain now, but this one seems to be quite
small so I hope its maintenance overhead would be low.

But I'd also like to have feedback from others. If you have a use case, or just
some time to play with something, giving it a try and coming up with ideas for
better interface, finding some of the bugs, etc, is welcome :-).

[spirit]: https://docs.rs/spirit
[tracing]: https://docs.rs/tracing
[log]: https://docs.rs/log
[dipstick]: https://docs.rs/dipstick
[tracing-dipstick]: https://docs.rs/tracing-dipstick
[tracing-subscriber]: https://docs.rs/tracing-subscriber
[example]: https://github.com/vorner/tracing-dipstick/blob/main/examples/shaving.rs
[instrument]: https://docs.rs/tracing/0.1.25/tracing/attr.instrument.html
[supported attributes]: https://docs.rs/tracing-dipstick/0.1.1/tracing_dipstick/#recognized-attributes
