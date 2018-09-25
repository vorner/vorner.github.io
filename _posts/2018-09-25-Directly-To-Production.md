# Directly to production after 6 hours of coding

This one is probably more of a personal story about Rust than the rest of the
articles here. And it is a half-in-joke (as opposed to the usual half-serious).

Sometimes, I write about the things that frustrate me about Rust. While I can
ramble about it for some time and with due passion, it's mostly a positive thing
‒ I'm frustrated by whatever language or tool I use. I wish things were better.
I question the sanity of its authors (especially if the author of the tool is
me). But this makes the improvements possible. And I write about the
frustration in Rust because I care. I've given up on many other languages and
tools as lost causes.

Anyway, today I want to share a positive story. It's not about 1000× speedup of
something (I've done that already with the [matrices]), or saving millions to a
company (I hope to do that one day). It's about a very small and mundane thing,
but I want to illustrate why it is I care about Rust.

## The background: the minimalistic music player

When I work, I like to listen to music. The old-style way, by having a
collection of songs on my disk, so it works even when I'm offline. When writing
a software that should run on a router and testing things out, this happens a
lot. When I travel, this happens a lot. I still have to find a hotel where the
WiFi works for 10 minutes without disconnecting. I don't really need fancy
playlists or collections sorted by the genre and the ability to rate the music.
I simply throw the whole set at the player, tell it to shuffle and then press
the `Next` button on the keyboard whenever I don't like the song.

Over the time, I've tried many full-featured players. `WinAmp` back at the times
when I still had Windows on my computer. Something with a name starting with `x`
that imitated `WinAmp` when I switched to my first Linux (at the times when
Fedora was still called Red Hat). One player that came bundled with Kde 3. The
other player that came bundled with Kde 3. The yet another player that came
bundled with Kde 4. I tried `mpd` and gave up in the middle of the
tutorial, telling me how to set up my music database and let it scan the songs
and how to move the songs from the database to a playlist (I guess this was more
of a documentation issues, someone told me `mpd` was in fact easy to use some
years later). This all felt too bloated and heavy-weight. I don't need no music
database, I just want to play songs. And I don't want to give the player 1GB of
RAM just to sit there (even now one of the machines I use is a cheap netbook
with only 4GBs, it's enough Firefox takes 2.5 of that for itself).

So eventually I wrote a program that got fed with a list of songs and listened
on a unix socket so I could bind a shell command that sends `next\n` into it to
a global shortcut in the window manager. The program used `mpg321` (or maybe it
was `aplay` at the time, now it is `mpv`) to actually play. The first version
was written in Perl if I remember correctly.

And, over the years, every time I was learning a new programming language, or
when the old version didn't feel just right, I rewrote it in the new language or
using some new trick. I could find a C version with threads, highly optimised C
version that `mmap`ped the file with a list of songs into the memory and
pre-indexed that so it saved RAM, Haskell version, Python version, another Perl
version I was using until recently… Well, I could find it if I really wanted to
go looking, but some of it might be still on some DVD backups in my parents'
house. In short, it became a kind of learning-something-new ritual.

Except for Rust. For some reason, I was using Rust for almost 2 years now,
considering it the go-to language for most of my needs (at least when nobody
else had any say in the decision) without rewriting the player in it. Until
this Sunday.

## So, how did it go?

I spend about 6 hours writing it. I tried using the 2018 edition for that and it
felt good (the biggest change for me is probably that I don't have to write
`extern crate` any more). I even used the much discussed match ergonomic thing
on purpose at some place and knew it at the time.

I used my own [corona] for the asynchronous stuff, to test it some more ‒ eating
own dog food and such. And also because this is the kind of thing the library is
for anyway: nobody cares about performance that much, with about 5 events per
second and 2 simultaneous connections, but it is more convenient that bare-bones
tokio or bare-bones juggling with threads and mutexes, and besides, application
that does close to nothing doesn't deserve more than 1 thread. Using more than
one thread without a performance need is *not elegant*.

I cursed a little when I discovered that I have to use either a thread local
storage or a mutex in a single-threaded application to store some mutable
globally-accessible object.

I cursed some more when I discovered the standard library or tokio don't even
consider the existence of unix named pipes, the mechanism the previous version
used to talk to `mpv` (by talk to I mean „pause“ and „quit“). A named pipe is
like an ordinary unix pipe, but it sits on the file system, so two independent
programs can connect to it (repeatedly). It needs to be created and then opened
like a file. But it acts like a pipe, so it can return `EAGAIN` (unlike files,
which claim to be ready to be read and written all the time even if they aren't
and then they block). And there doesn't seem to be any obvious way to open a
file and then plug it into tokio and tell it „treat it somewhat like a socket“.
Thinking about it, tokio probably has only unix domain sockets, but not
unidirectional pipes at all. I mean, I could put something together with all
these [`AsRawFd`] traits and [`Evented`], but:

I had to spend about an hour and a half, debugging a lockup, blaming it on Rust
not knowing what to do with pipes and playing with `strace` only to discover
that even *opening* the write end of the pipe blocks until there's the other
side ready to read it and that the Perl version launched the `mpv` first and
only after that opened the pipe (I guess I know why now ‒ I have the feeling I
might have spent several hours on that when writing the Perl version back then
too ‒ reminder to use the great invention of code comments sometimes). And then
I spent another 10 minutes switching to unnamed unix domain socket pair and
screwing that one onto the file descriptor 4 of the `mpv`-to-be to be done with
the damn pipes.

## So, what's the big deal?

If you read the above, you might think that I could have written it with the
same amount of time and cursing in any other language I know. And it's probably
true. Well, maybe not `make` and `bash` might be a challenge too (hmm, thinking
about that… 😇).

However, in every previous version, there was something, some nudging feeling,
that something in the code is a bit *off*. Every previous version was glitchy
for the first two weeks of using and I had to fix it several times during that
period. The previous Perl version had a bug I never managed to find ‒ sometimes,
the program just terminated. No crash dump, no error message, it just gave up
and I don't know why. Every previous version felt like a temporary solution
until the next attempt.

No previous version ever had the prev/next navigation entirely correct, I always
had some scenario of going forward and backward through the history where a song
got played twice in a row or skipped. One of the previous versions sometimes
started to play two songs *at once* for no apparent reason. Having to express
myself in Rust's `Option`s and encoding the state in the forgiving-nothing type
system made me think about it long enough to do it right this time, in a way
Perl's „everything quacks like a string“ attitude never would (or C's „… like an
int“).

The Rust version went straight „to production“. From the Sunday evening, I
discovered no glitch and the thing feels *finished*, at piece.

I mean, what program *ever* feels finished? And I'll probably do some thing with
it eventually (I'm already thinking about keeping the list of song files in
memory as a compressed inverse radix tree to save some of the RAM ‒ not that I
would have any need of that), but I wouldn't have to. Whenever comes the time
someone makes a better language than Rust and I move to it, I'll have a hard
time finding the *reason* to rewrite it in that language. The bye-I-give-up bug
was bothering me for ages, but eventually it was the excuse for the rewrite.

And this feeling ‒ the feeling that I think the program is *correct*, despite
plenty of experience telling me no program is *ever* correct, is the reason why
I care about Rust and why I like to use it. That I can finish the program, try
it once or twice, put it into production and forget about it. Yes, it is longer
than the Perl version. It probably took more effort to write. But hey, it's
worth it (and besides, it isn't *that* much more effort).

Wouldn't it be great if we were able to write programs the admins have to touch
only when the hardware the programs run on goes to the silicon heaven and they
need to install them onto new servers? Rust gives me hope this dream might one
day be possible.

## The result

You know, this is not about the destination, it's about the journey. Or mostly.
At least for me.

Alright, if you insist, the code lives in the [github repo]. Go find that bug
and tell me it is *not* correct, if you have to.

It doesn't contain the shell commands to control it, though. ~~I was too lazy to
take them out of my system and import them~~ I thought it would be a nice
exercise for the reader to reverse-engineer them, they are quite simple.

Anyway, it might be less work to spend the time writing your own instead of
trying to integrate this into your own work flow and window manager.

[matrices]: 2018-05-12-Mat-perf.md
[corona]: https://crates.io/crates/corona
[`AsRawFd`]: https://doc.rust-lang.org/std/os/unix/io/trait.AsRawFd.html
[`Evented`]: https://docs.rs/mio/0.6/mio/event/trait.Evented.html
[github repo]: https://github.com/vorner/playlist_mgr
