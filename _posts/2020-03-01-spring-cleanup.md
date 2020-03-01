# Spring cleanup

You may have noticed this blog (is it still fashionable to call something a blog
or am I 15 years out of phase?) was silent for a while. The reasons for that are
multiple, I'm going to blame mostly two things. One, the
[Factorio](https://factorio.com) game. It's a great game, which is the actual
problem ‒ it demands *a lot* of time. The latter is some kind of feeling of
being tired. I suspect this is more about both my day job and being winter time
(are there actually people who don't need 14 hours of sleep a day during the
winter?) than Rust itself, but I was still thinking about what the reasons are.

What I've noticed is, the whole ecosystem feels somewhat less vital and more
tired than a year or two ago. It might be just my perspective. But it might also
be something related to this idea that programmers come in two flavors ‒
Conquerors and Stewards. Unfortunately, I seem unable to find the original
article now (if you know where it is, please let me know, I'll link it here),
but the gist was that Conquerors like to explore new things, move fast, break
things and they tend to leave some unfinished mess behind when they move onto
something new. Stewards like to nurture the code base. Both are valuable when
applied to the right problem.

So if you're a Conqueror, you're likely to build a library, maybe give it
reasonably good documentation, throw some examples in, keep it maintaining for a
while. But once you release something like `0.3.14`, you notice that it's no
longer as much fun because there's nothing interesting happening and you move
onto something little bit different.

If you are a Steward, you care about the day to day quality of life of the
library. You make sure the CI is in great state. You take care that every new
contributor feels welcome. You improve the documentation and fine-tune the API
so it feels fully natural to use.

While I'm probably not a 100% Conqueror ‒ I don't like leaving mess behind, I
still prefer the exploratory phase. I think a lot of libraries that feel
somewhat unfinished and abandoned over the ecosystem might be result of
something like this, lots of people around the younger Rust are more of a
Conqueror.

## Enough of the chatter

This isn't supposed to be only philosophical rambling. I've came to conclusion
that it's not healthy for me to pretend to myself that I'm maintaining some 20
crates while it's not really true. I wanted to admit to myself which libraries I
intend to keep even as a Steward (polishing, bringing them to 1.0, …), which I
want to hand over to someone else and which ones are probably just dead already.
I want to do some spring cleanup of „my“ libraries.

The reason I'm writing it publicly is twofold:

* I want to force myself to actually go through the libraries and at least state
  the reality in the readme or the maintenance badge. People looking for a
  dependency should know its state.
* This is open source world. If you don't want some of the libraries to die,
  *it's your opportunity to do something about it*. See the bottom of the post.

## Libraries I want to keep and invest my time in them

I have limited amount of time and energy, but there are libraries I feel I have
enough emotional bond to want to keep them myself. That doesn't mean you can't
help me with them (I really like people taking interest, it's one of the few
rewards I get for doing the work). But I still want to consider them *mine* in
the sense I want to be the one doing reviews, having opinions and being proud of
them, as well as *mine* in the sense of being responsible for them being
high-quality and working.

* [arc-swap](https://crates.io/crates/arc-swap). This one introduces
  something like atomic `RwLock<Arc<T>>` and few other related utilities. It's
  one of these things that look much more interesting below the surface. I'd
  like to do some API polishing and then move it to 1.0 during the next few
  months.
* [signal-hook](https://crates.io/crates/signal-hook). Safe interface to Unix
  signal handling. I'd like to go 1.0, but there's this thing about Windows. I
  don't have a Windows machine and I don't really want to own the Windows side
  of the thing. From time to time someone comes and declares they'd add Windows
  support… and eventually disappears. So I'll probably just go ahead with
  Unix-only 1.0 for now.
* [log-reroute](https://crates.io/crates/log-reroute). Ability to change the
  logger of the [log](https://crates.io/crates/log) multiple times during the
  lifetime of a program. It's small and I actually use it now and then, so I can
  as well keep it. I might move it to 1.0 eventually too, as there's really
  nothing much of it.

## Things I'm not sure about

I probably won't let these die, at least not for now. But if you want to take
interest in them and take over, I'm fine with that. If there's a bug, I'll
likely spend the time fixing it, and will do so reasonably fast.

* [err-context](https://crates.io/crates/err-context). It's a very minimal way
  to create multi-layer errors and handle dynamically-typed errors without
  forcing a specific error type or library onto users on the other side of API
  boundary. Together with [err-derive](https://crates.io/crates/err-derive) it
  creates possibilities of the [failure](https://crates.io/crates/failure)
  crate, but without the downside of forcing everyone to agree on their error
  handling strategy. It's created mostly because the right strategy of Rust
  error handling is just endlessly being discussed and I wanted to handle errors
  *now*. It's not perfect, but it gets the job done. On the other hand, it feels
  like I'm the only user of that thing and there are like few millions of other
  error handling libraries, so I'm not really trying to make it popular and I'll
  just wait what happens there.
* [contrie](https://crates.io/crates/contrie). A concurrent lock-free map
  implementation. It turns out the data structure this is based on is not that
  great in practice and the practical use cases are kind of very niche. I've
  also tried to just lay the foundation and let the rest of community do the
  polishing, as a kind of social experiment ‒ and I don't think this went
  particularly well. I might be doing some kind of PR or marketing the wrong
  way. Or nobody needs this thing. On the other hand, it was fun writing and
  there probably can be more exploration around it.

## The victims

These I plan to spend as little time on as necessary. Which probably means
mostly doing reviews and releases if they come, but I don't expect a lot of them
would be coming. If you report a bug on them, be prepared to send the code of
the fix too. I'm willing to invest all the needed effort for a transition,
though.

* [corona](https://crates.io/crates/corona) (no relation to the virus spreading
  right now, the name is play on coroutine and I had the light around a solar
  eclipse in mind at the time). This was a somewhat hacky solution to make
  async-await interface for Rust at the times of
  [futures 0.1](https://docs.rs/futures/0.1.*) and
  [tokio 0.1](https://docs.rs/tokio/0.1.*) (actually, it goes back to the
  venerable [tokio-core](https://crates.io/crates/tokio-core)). It uses
  stack-full coroutines and doesn't support work-stealing schedulers. It's been
  fun writing, but nobody ported it to the new futures. And I feel there's no
  *need* to port it ‒ it played its role when waiting for native `async/await`
  notation, but there's no motivation for it now that we have it. On the other
  hand, porting it to the current ecosystem might be interesting and possibly a
  bit advanced learning experience for someone.
* [spirit](https://crates.io/crates/spirit) and the whole huge family of related
  libraries (including [structdoc](https://crates.io/crates/structdoc)). I still
  feel some library like this is needed and I still don't see any other suiting
  my needs (specifically, I don't want a *framework*). But I just don't have the
  time to keep the whole thing alive on my own, the laundry list of tasks that
  needs to be done to just keep it alive and up to date is huge. I feel like
  some company using this might want take the burden of developing it. In case
  someone finds the time to work on it, I might find my motivation again too and
  help (or at least guide lost ones through the code base; there are some um…
  *interesting* parts).
* A lot of small things with like 20 downloads per year. These are likely mostly
  dead, but I'm fine with someone trying to resurrect them.

## If you want to help

As mentioned above, help is appreciated. If you feel you want to keep something
of these alive or that you'd be a good steward for a library, let me know ‒
issue on github is fine, an email is fine too.

We would agree on the specifics on case by case basis. But my general view on
this is that I first want to somehow trust the person (Sorry, that's not
personal, I do want to believe in good intention of people, but in the world
where we have library takeovers and malicious code injected in them, there
should be some kind of due diligence. I also want to know such person won't
disappear a month after handing the code over.). And you might want to try it
out first too, to see if it's your kind of thing. Therefore I imagine some kind
of gradual shift ‒ first seeing and reviewing some improvements, then handing
over rights to merge things, etc.

If you feel like you'd want to try, but you're not sure if you're up to it, then
I'd say you should go ahead. It's not like there would be people queueing over
wanting to maintain something so you're not taking someone else's place and the
library would likely die without you. And I'm not going away completely, so I
can help out.

In general, as Rust matures, I think the community has higher need for Stewards
(as opposed to the time when Rust was young). Do you feel like being one? I'm
sure I'm not the only mostly-Conqueror who would like to hand the „boring“
details of running a proper civilization to someone more interested.
