# Throw away code

There's an ongoing discussion about what makes Python better prototyping
language than Rust (with Python being probably just the archetype of some
scripted weakly-typed language). The thing is, I prefer doing my prototypes in
Rust over Python. Apparently, I'm not the only one. So I wanted to share few
things about what makes Rust viable for these kinds of throw-away coding
sprints, at least for me.

## Our goals

Sometimes, our goal isn't really to write perfect code that is performant,
correct, handles all kinds of errors sanely, has great UX and is maintainable.
These projects are what we are proud of, sure. We pin them on github profiles.
We write blog posts about them. We write whole handbooks of best practices how
to do them.

But sometimes we just need to throw something together really fast and don't
care about the quality as much. That kind of bit of cardboard and huge amount of
duck tape thing. These include:

* Single-use debugging tools („I need to throw 10k of these weird requests at
  the server to see if it triggers the bug. It didn't? Ok, let's try something
  else…“)
* Searching for a counter-example to a claim in a scientific paper („I can prove
  it's a counter example once I have it, so I won't need the code any more“)
* Processing bunch of data just once („I wonder how many of these `.txt` files
  have broken unicode in them“)
* Figuring if something has any chance to fly at all, before committing to it
  („Could I distribute the changes as compressed binary diffs, or would that be
  too large?“)
* Demonstration purposes („We would like to build something in lines of this,
  but, you know, actually working“)

Of course, there's a lot more. I'm not even sure if there's more of the „proper“
coding or of this „throw away“ coding. Except that we don't really brag about
our throw-away code („Look what terrible monster I've stitched together during
the lunch break“), we don't write tutorials how to write them much, etc. So this
is exactly the kind of blog post we don't write 😈.

Instead of writing something proper this time, we are going to talk about
how to write terrible code, but fast. We have decided to make an explicitly
sloppy job of this one and admit it to ourselves not to feel ashamed of it
later:

* We want to spend as little time on it as possible. Just do it, throw the code
  away after it had its use and move on. This one is going to be over by lunch
  time.
* We don't care about performance that much (as long as it finishes running
  before the lunch too).
* We don't care about handling all corner cases, only the ones we actually
  encounter in the data.
* We don't care about documentation or readability.
* We don't care about tests, provided we are confident enough the answers are
  accurate enough.
* Actually, we don't really care at all…

*Note: make sure not to let anyone put this into production 😇. If you don't
delete it, make sure there's a comment on a prominent place warning people not
to use it.*

## Why do people think Python fits here better than Rust

The thing is, Rust *makes us* care. That's one of the points of Rust. It'll
complain that our code is not production quality and that we need to do better
to save on the pain down the line. Its type system can be a real prick in
insisting on little details, like that ints and strings are not really the same
thing and that there's a difference between owned and borrowed thing and… Well,
you know, all that stuff. Rust wants you to make good, proper, maintainable
code.

On the other hand, Python doesn't really insist on anything. Therefore, it is
easier to not care in Python.

## My own experience

I know a bunch of programming languages and reach for the one that I hope would
suit me best in the given time. So, for some really simple things I simply put
together few lines of shell (and some slightly less simple ones ‒ I'm ashamed to
admit that some 1000 lines long shell monster kept running in real production
for years ‒ but it *did run*). If it can be done by 2 or 3 ugly pipelines, it's
fine.

Over the years, I've used Perl a lot (that one doesn't care if it's int or
string… no, correction, in Perl everything is a string, ints just don't exist.
Well, kind of). It's probably *the* language designed for throw away coding.
I've done some Python too (that's like Perl, but with proper objects in it, and
everything is a dictionary there).

But recently I've noticed that if I try to do a similar thing, I do it faster in
Rust. Not that it runs faster (well, that usually too, but that's not the
point), but that I'm done with the task at hand sooner and with slightly smaller
amount of cursing.

This certainly is in part because I'm more proficient in Rust than in Python.
It's also because the Rust mental model is closer to how my brain works than the
Python one. **Your mileage will vary** ‒ if you're a Python matador who's been
coding in it for decades and are just learning Rust, you'll certainly do it
faster in Python.

But also, there are some tricks you might employ to do these things in Rust
faster (that is, faster than you do now, not necessarily faster than in
`$OTHER_LANGUAGE`).

## Tricks for faster coding

### Compile times

Rust is known for its slow compile times. Python has *no* compile times. If you
have to wait every time for the compilation just to have a bunch of errors
thrown into your face, it's going to slow you down. Especially because Rust
*likes to* throw bunch of errors at you every time you try to compile it. Rust
is known for its great error messages, so it wants to brag how good they are by
using them *a lot*.

You can, however, notice that you don't really need to *build and run* every
time. That you often just want to check everything is on the right path. For
Python, you do need to actually run the thing (because Python doesn't really
have much of a compile time so it likes to throw the bunch of errors into your
face at *run time*), but Rust is the language that „if
it compiles, it's correct“. And by complies, I actually mean mostly type-checks.

What does this all mean? You can check out:

* The `rust-analyzer` language server. You'll be getting red squiggles in the
  editor instead of having to compile. It's not perfect (sometimes the list of
  errors is different, sometimes it just gives up on that particular project),
  but it's getting better and it points out most of the errors without any
  compilation at all.
* `cargo check` performs just the first stages of compilation and will stop
  before codegen. It means it doesn't produce anything that could be run, but
  it is so much faster and provides the bunch of errors we so much want to have.
* You can let `cargo watch` keep recompiling the code asynchronously in another
  terminal. I just glance at it to check if there are any errors around, but I
  don't wait for it ‒ at worst, the list of errors is one iteration outdated. It
  can be used for other things, like keeping the documentation of the current
  crate up to date, or having a head start at compiling the executable, or even
  having all the tests being re-run on each save (I'm getting off topic here; we
  are being sloppy here on purpose, so what tests are we talking about?)

These don't make the compile times shorter, but it eliminates the *waiting* for
them from the hot coding path. It still takes some time to compile (especially
if you have a lot of dependencies and do a clean release build), but that
doesn't mean it has to slow you down.

### Embrace the type system (and borrow checker and all of these things)

After some time working with Rust, one learns to lean onto them instead of
fighting them.

This is where most of my own speed up comes from and what I miss about Python.
When I want to know if my code is working, I actually have to run the Python
thing and feed it with data. Which means I either need to set up a smaller input
or wait for the whole thing to get crunched, only to have it explode on some
typo or switched order of parameters after 5 minutes of running. After 10
iterations of running the Python code (each crashing later and later in the
code), it finally finishes. By that time, I'm no longer confident it does what
it should, after all these retries, so I go back and have to figure a way to
double-check it.

In Rust not only I don't have to run the code until it is almost finished and
even when I feed it the whole input (which I usually do), it's usually faster
and it runs to completion the first time. I also can move through the code much
faster. With Python, I stop to check the documentation, think about what type
goes where, etc, exactly because it's so painful to find out only at runtime. I
need to be careful while writing the code.
With Rust, I just type the code, get the red squiggly, fix it and move on. I
outsource that effort of checking if these things click together in any
meaningful way to the compiler.

This is kind of in the theme of „hurry slowly“ approach. By making sure
everything has the right types and aligns well, it makes each iteration slower.
But it also makes it possible to have much fewer iterations before the whole
thing works well enough.

Also, don't fall for the impression that throwing `unsafe` in there to bypass
some of the checking will save you time. It won't. It's a trap. If you don't
know for sure that you need it, then you probably don't and doing `unsafe` right
is a lot of work. Doing it wrong is easy, maybe easier than doing it safely, but
you'll pay for it later on, when trying to figure out why the thing does
something arcanely weird. If you put any non-trivial `unsafe` in the code,
you're risking spending days and nights in front of a debugger. The checks are
there for a reason.

### Take the easy way out

I don't say to clone everything. Even in prototyping code, I often take `&str`
as parameter if it's just „looking at it“. But I do so in the obvious, trivial
cases. The ones I don't really need to think about any more.

But if you ever find yourself thinking about writing any kind of
`-> impl Iterator<Item = &impl Display> + '_`, just stop and throw a
`Vec<String>` in there. Where you would care about limiting the number of
allocations or conversion in true production code, or would design the right
schemes to borrow things, shamelessly let the computer do a bit more work to
conserve the time and mental energy. Don't feel bad about not putting lifetimes
everywhere, or `Arc`s or `RefCell`s if it makes you move faster. That may mean
the design is sub-optimal, but if it works…

After some time of Rust coding, you get the itch of „this really should be
possible without this one particular heap allocation“. Nurture that intuition,
it's useful one. But when on a speed run, make sure to set it aside.

But also do listen to Rust telling you the design sucks too much. If the code
gets too infested with `RefCell`s or other smell indicators to the point where
it's impossible to see through it, it's a signal it's not well thought through
and that you might not know what you're trying to do. Sure, you could force your
way through all that, but it would probably produce the wrong results anyway.
The goal here is not some abstract elegance, the job needs to be done fast, but
it's often faster to make sure one understands the dealings first than to write
it without the understanding. Rust seems particularly good at hinting at the
places where the design is just lacking.

But of course, the balance here is different between production and throw away
code. Let yourself wade through some amount of `RefCell`s and `Rc`s, especially
if you see *why* the design is suboptimal but it's still less work to write. The
trade off is different for this and for a code that's supposed to be around in 6
months or 6 years.

Other shortcuts one can take:

* Relax with the error handling. I personally avoid `unwrap`s for user errors
  even in this kind of code, but don't hesitate to just put `anyhow::Error`
  anywhere and let it bubble out of `main` (I want to see at glance what I think
  is a bug in the code and what is just me giving it non-existing file name).
  Sometimes, I attach a context to it (like the file name it is processing)
  because I expect to mess up something in around there and knowing the file
  name will save me some time in the whole session, but not too much. I expect
  to know the code when using it, so I won't need too many details ‒ after all,
  it's going to be short and used just 5 minutes from now. The error messages
  don't have to be perfect and I'm just not going to design my own error types.
* Don't overdo it with performance tuning (both CPU and memory). Will it make
  your life easier to load the whole file into a `String` by
  [`std::fs::read_to_string`](https://doc.rust-lang.org/std/fs/fn.read_to_string.html)
  instead of reading line by line? Sure thing, go for it. If it doesn't fit
  into RAM, still prefer
  [`.lines()`](https://doc.rust-lang.org/std/io/trait.BufRead.html#method.lines)
  over the faster option of manually looping over
  [`.read_line`](https://doc.rust-lang.org/std/io/trait.BufRead.html#method.read_line)
  with reusing the allocation.
* Allow yourself not to care about warnings. Does clippy complain about
  `unwrap_or` being slower than `unwrap_or_with`? Let it complain. I still keep
  the warnings on. In part because I'm lazy to turn them off, but if the thing
  doesn't work in the end, warnings are the first thing to check if there's
  something fishy. Just mentally tick them off as harmless and move on, don't
  spend time refactoring and rewriting the code to silence them.
* Log or print stuff all over the place. It doesn't have to be perfect, tunable,
  or whatever. But having a way to check that it did or did not run well is
  nice. And debugging by these `dbg!` macros, if they are already in, is usually
  faster than running the whole thing again in a debugger. You can also use them
  for eyeball-level benchmarking (does that part feel slow, or does it print its
  line fast enough) and you don't need any better benchmarks than these.
* Don't bother with `async/await`. You're not handling millions of
  concurrent connections in that prototype. If you can't just handle one
  connection after another in a sequential manner, launch threads (and forget
  about them/log if they error out).
* Use indexes into a big `Vec` instead of references in complex data structures.
* Use simpler algorithms. Do you need to compute a median? Sort the array and
  pick the thing in the middle. Is that optimal? When the measured metric is
  developer time, sure it is.
* Don't hesitate to generate code ‒ use existing derive crates, write simple
  macros (*simple* ones, though, don't bother caring about them working in all
  possible situations, your copy-pasting spree is what matters), or write a
  `sed` command that spits out something close enough to Rust code that can be
  manually edited to compile by changing 5 characters here and there.

### Know when to step back

If on a short time budget, it's important to remember to not keep banging the
head against one single problem for too long. If you get stuck on the same
lifetime issue for more than 5 minutes, decide you should abandon that
particular road. You don't need to win this particular fight. The API doesn't
need to be perfect, or whatever. Just `Rc` it or something like that. Or just
decide that part of code is really not necessary at all 😇.

### Know the stack

Both the standard library and the crates out there contain a lot of goodies that
can make you move faster. Give them a read, explore. This can be done during a
low-productivity time. Instead of browsing the facebook during the train ride or
after lunch, one can browse the methods on
[`Option`](https://doc.rust-lang.org/std/option/enum.Option.html),
[`Iterator`](https://doc.rust-lang.org/std/iter/trait.Iterator.html) and
[`Result`](https://doc.rust-lang.org/std/result/enum.Result.html). Did you know
that you can
[`Box::leak`](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.leak)
to get a static reference? Similarly, there are some useful crates that are
worth checking out and can make the life easier (naturally, the list is
incomplete):

* [`anyhow`](https://docs.rs/anyhow) (or something similar) for errors.
* [`itertools`](https://docs.rs/itertools) for extra methods on iterators.
* [`rayon`](https://docs.rs/rayon) which often lets one to speed things up almost
  for free. Depending on how fast you need to be, but if you can make it run for
  1 minute instead of 5, it can speed up your iteration process and it is worth
  it if the only change you need to do is put `into_par_iter` at the right place
  in the code.
* [`bumpalo`](https://docs.rs/bumpalo) is a bump allocator. While the main
  motivation is performance, it also may make your memory handling and lifetimes
  in complex data structures much easier.
* [`once_cell`](https://docs.rs/once_cell) if you need global variables that
  need some initialization.
* [`crossbeam_channel`](https://docs.rs/crossbeam_channel) and
  [`crossbeam_utils::thread`](https://docs.rs/crossbeam-utils/0.7.*/crossbeam_utils/thread/index.html)
  make it easier to write things like producer-consumer patterns. Do check that
  you need them, though. Oftentimes, just producing everything ahead of the time
  into a `Vec<_>` is good enough.
* [`StructOpt`](https://docs.rs/structopt) if you need command line.
* [`env_logger`](https://docs.rs/env_logger) if you need to produce logs
* [`serde`](https://docs.rs/serde) if you need to serialize or parse some common
  structured data formats.

The idea is to have the feeling of „I think there was something around there
that did just the thing I need right now“ and know where to look for it, not
necessarily remembering each detail about them.

If you need something else, make sure to check if there's a crate for it. These
5 minutes of searching can save hours of fixing the hand-rolled CSV parser. The
ecosystem contains a lot and during these prototypes, it's often acceptable to
use an outdated unmaintained half-unfinished dependency (while one would be very
reluctant to put that into production code, of course). Pick the one with good
documentation and convenient API, not the one with most impressive benchmarks.
Pull dependencies in instead of rewriting the wheel once again.

## Recap

Depending on how proficient one is in either language, Rust can be a viable
option when writing one-off bits of code. It does however require a mental
shift. The way one writes these throw away codes is different than the way one
does it in Python or Perl. The goal here is to take shortcuts strategically, to
know which things just don't matter. Make sure to streamline the general work
experience. But in the end, the optimal result would be to write it correctly on
the first try (even though this first try is going to take longer) than to
iterate many times, like one would do with the scripting languages.

Also, it needs a mental shift from the usual production quality Rust coding ‒
the one that's described in all the tutorials and books. Keep in mind that the
optimization metric is developer time and make strategic decisions based on
that. Make sure to keep it in mind that the maintainability doesn't matter, that
running 4 times slower doesn't matter and that you're allowed to take shortcuts
as long as they won't have time to bite you during the speed run.

Of course, some of these techniques are not exactly Rust specific, but they need
to be reminded in context of Rust, which doesn't exactly recommend them.

And, by the way, don't forget to ask if you actually need to code anything at
all, or if there's a whole already existing tool doing just what you need, or if
two `grep`s and one `sed` will get the job done.
