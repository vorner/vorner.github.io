# Passerby contributions

Before Rust, I didn't contribute much code to projects over the Internet. Not
that I wouldn't like opensource or that I wouldn't want to help out, I just
somehow never found a project I'd like to fully join and keep working on.

I still haven't find a project I'd like to join. I do have few little crates of
my own and I keep them alive, mostly because nobody else would. But I now
contribute much more code than previously, I just do it across a very wide range
of projects. I call these a passerby contributions ‒ I walk by a library, try to
use it for something and discover a papercut. An hour or two later, I'm opening
a pull request to the crate. I might open two more in total and then move on to
the next library.

So, while Rust didn't help to solve my problem with settling down with a
specific project, it at least allowed me to compensate for it somehow. Let's
look at what I think helps.

## Friendly community

For each individual project, being friendly is more important for *keeping*
contributors than getting enough of the one-time ones. However, the bigger Rust
community somehow includes all the small communities of individual crates and I
tend to expect them to be friendly too. And mostly they do keep up with the
expectation. Telling me I'm wrong is totally fine. After all, it's not *my*
project I'm trying to modify. I just don't like the feeling of outright not
being welcome I got in some other (non-Rust) projects. Even if someone is
telling me to RTFM, accompanying it with a ling to *which part* is a big help.

In the same area, response time on the pull request is a *huge* part of the
feeling. I certainly have more motivation to send another pull request (maybe
not right now, but in few weeks or months) if I get a fast feedback. Not
necessarily getting it merged right away (though getting it merged *and
released* in next 2 hours is a huge motivation boost), but even a confirmation
the maintainer is alive and willing to have a look is a plus (eg. „I don't have
the time right now, I'll get to it sometime next week“). I have much weaker
motivation to fix things on projects where my previous pull request still rots
for several months without any answer (that, unfortunately, happened even for
some rust projects).

## Getting to the coding fast

If you're working on some project at least semi-regularly, you don't care about
this because you're already set up. But when I considering fixing a typo in
the documentation in a project I never contributed to, I need to do quite
a few things first:

* Locate the right repository of the project.
* Get it to your own computer.
* Figure out how to build the project.
* Manage to run at least a basic subset of tests.

Rust tooling cuts on this effort a lot. The [crates.io] has a link to the right
repository. I don't have to search for it, choosing the right one from 5
different places (two of which claim to be the official one).

And *usually* building and running tests is just the usual `cargo` stuff. With
many projects, figuring out building is for hours. Not to mention some projects
that have just *outdated* documentation about how to build them that fails, but
only at the end of the very last step and one is *supposed* to just know the
project is using `cmake` instead of `autoconf` for *ages* now (but the
documentation still says `autoconf` and the files for it are still present in
the repo).

## Getting the bearings through the code

It is said that a code you wrote 6 months ago could have as well been written by
someone else. That's only half a truth. Maybe you don't remember the fine
details of the code. But it'll feel familiar. And if you've been working on a
different part of the same project, you'll be used to the same conventions. If
presented with the same problem or design decision, you're likely to come up
with several, including the one actually in the code. It isn't a code you
outright know, but it's a close relative to it.

Let's say I get to a project where I don't know the conventions. I start from
scratch and know only what I read from the code right now, not what I remember
from before. I can probably do some *basic* assumptions ‒ like programmers are
generally lazy and prefer the obvious and simpler solutions and that they don't
*actively* try to make my life harder by choosing [misleading names][intercal].

So, how does Rust help with that? There are actually several things.

First, cargo helps keeping the code in bite sized chunks. When it's easy to use
a lot of different libraries, libraries tend to be small. So even if I had to
read the whole thing to understand it, the whole would be rather small. In the
C++ world, it's a big hassle to include a library into a project, so libraries
tend to be *huge*. Of course, there are exceptions to this ‒ [rustc] itself is
quite a big piece of code and it's much harder to get oriented in it. It's still
better than other code bases of that size, though, thanks to the other things.

Second, Rust have relatively consistent conventions. Most things don't digress
from the `std` style much. If you want to make life easier for contributors (and
users of your API too), stick to it. There are even [style guidelines].
Unfortunately, it seems to be abandoned and incomplete ‒ but the things in there
are worth the read.

And finally, Rust encodes a lot of information into the code other languages
have only in documentation. People say that the Rust ownership system is
novel. It is not. If you write anything larger in most any language, but
especially in languages with explicit memory management like C or C++, you
*will* end up with some equivalent of ownership and borrowing. Rust only makes
it *explicit* in the code instead of the comments (and checks that it makes
sense).

This goes for many things. With Rust, I don't have to study the comments to know
what variables are protected by which mutex or that this return value only
indexes into some other buffer and is useless without it. In python, I have to
look through the code for the callers to see what types are passed as
parameters to the function I'm so eager to edit (and I'll still wonder if
someone somewhere else passes something only *similar*, but slightly
incompatible with the change I want to do and I just missed it). In Rust, I have
to look at its signature only and I can be 100% sure it's the only type that
will be passed in.

The „line noise“, as many refer to the numerous `&` and `?` and `mut`s around
the code is a big part of this. Many beginner Rustaceans ask why it is
necessary and if the compiler couldn't figure it out by itself. The answer is
that the *compiler* could. You can ask the compiler to please read all the code
and figure *a lot* of things for itself. The line noise is there for me (and you
6 months from now), someone who isn't familiar with the code. It's documentation
of the author's intentions. Because I'm *not* going to read all the code (and
certainly not remember it perfectly even if I did), I'm going to read the 10
lines inside the function I'm editing and *maybe* the struct definition this
function manipulates. Thanks to the line noise, this is *enough*. The
information compressed there contains warnings about places where something
might modify some data or where the control flow might early return on me. This
doesn't help only in passerby pull requests, but in large code bases too. In a
large code base, you're dealing with code you don't know *all the time*.

And I'd like to bring attention to the fact that natural language has an
equivalent of the line noise too. It's different for different languages, but
let's pick the articles in English. If I have a sentence „Could you pass me a
hammer?“, the „a“ in there clearly states that I don't really care about
any special properties of hammers. If I instead said „Could you please pass me
the hammer?“, I'm stating that I do care about a specific one and that you're
already supposed to know which one (if you didn't, you'd double-check by asking
which one, while in the first case you're just going to grab the first one you
find). If instead I said „Could you please pass me the big hammer?“, I'm stating
that there's another smaller hammer around somewhere and I want the bigger of
the two.

# In conclusion

I often wonder how many bugs are eradicated by the safety guarantees of Rust and
how many simply by making sure all the important details are right in front of
the eyes of the programmer.

I also wonder if this in turn could in some way help with having the more
friendly community. If I don't have to spend the time digging for details I
*could have* in front of my eyes, I have a better day and I certainly am in a
friendlier mood to others than after a day of hectic cursing the bugs caused by
missed details.

Anyway, thanks for making Rust easy to read (even if learning to read might seem
difficult at first). It certainly makes contributing more fun and lowers the
entry barrier.

And if I ever sent you just one pull request and never came back, don't worry.
You haven't chased me away. I just don't know how to stay at one project.

[crates.io]: https://crates.io
[intercal]: https://en.wikipedia.org/wiki/INTERCAL#Syntax
[rustc]: https://github.com/rust-lang/rust/
[style guidelines]: https://doc.rust-lang.org/1.0.0/style/README.html
