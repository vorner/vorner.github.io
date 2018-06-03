# Design guidelines: General principles and project statement

There were some heated discussions in Rust community as of late. During that
discussions, I argued that some best practices for RFC authors would improve
both on the results as well as the discussions and I promised to give it a try.

However, I'm not an RFC author myself (a suggestion to include stabilization
FPCs in TWiR doesn't really count). Therefore, my authority to write about how a
good RFC looks like is severely limited. So I thought I'll write a blog series
about designing things. I do quite a lot of that and RFC is just a special case
of design.

In the long term, I'd like to consolidate the blog posts to something more.
There's a lot of learning resources about Rust:
* [The Rust Book](https://doc.rust-lang.org/book/second-edition/index.html).
* [The Rustonomicon](https://doc.rust-lang.org/nightly/nomicon/).
* [Rustc guide](https://rust-lang-nursery.github.io/rustc-guide/) for hacking on
  the compiler itself.
* A lot of others I forgot to list.

But there's not much on design ‒ on what you do *before* you start writing the
code. That phase is probably more important than understanding some
syntactic tricks.

So I'd like this into grow to some kind guide (I don't know *exactly* what kind
yet). But I'd like input from other people on this. If you are interested in
helping out, contact me on vorner@vorner.cz ‒ we can talk about some form this
could have and how to go about it.

For now, I have some of my own personal observations what worked for me. It is a
collection of things I learned from the Internet, several books and practice (I
can't claim much originality here). And what works for me doesn't have to work
for you. If you have something that works better ‒ well, see the above
paragraph. Also, as with any guidelines, it is appropriate to go against them
from time to time. Use your own common sense. And it is certainly not a silver
bullet ‒ it might help you improve, but following it isn't a silver bullet to a
good design.

## General principles and attitudes

Oftentimes, there's a lot of non-technical stuff around design. Emotions.
Misunderstandings. Politics. Usually, I try to avoid them as much as possible,
but they are still there. My goals are usually to have a good solution to a
problem, not gaining points in corporate power struggles. So, good design is as
much about the technical aspects as about psychology, negotiations and keeping
one's sanity.

I'm often hard to argue with, both because I have naturally stubborn nature and
because I usually give some thought to the design upfront. It takes good
arguments to prove me wrong, but it can be done (and it is done quite often) and
I actually welcome that. And I don't *like* arguing ‒ there are just some times
where I'll do it because I believe it's worth it. I'll avoid it if I can.

So, if you find yourself arguing with me, remember it may be understood as a
form of respect in itself. And here are few tips how to do so. I try (and
sometimes fail) to follow them myself.

* Optimize for reading and understanding, not for writing. Whenever you write a
  proposal, it'll be read by many more people. It saves time many times over for
  them if you are clear in what you wrote. And it saves *your* time if you are
  not misunderstood and you don't have to explain again. Misunderstanding is one
  of the biggest cause of arguments, wars, bad designs and unhappiness. Of
  course, similar things apply to designs discussed in person or in other
  non-written forms.

* Try to avoid personal attachment to your code, your genius solution, your
  precious ideas. Remember that you ‒ that we all ‒ strive for a *good*
  solution, not necessarily *your* solution. If your proposal gets replaced by a
  better solution, your solution played a much needed role in that. It was an
  ancestor to that good solution. Try to build a solution that is not merely
  *yours* but the *team's*. It's good for the result, but also for feelings of
  everyone, which is no less important.

* Don't force anyone into a defensive position, unless you really have to.
  People are much likely to agree with you if that doesn't include admitting
  they are *wrong*. Better off, don't even *think* they are wrong. Allow them to
  „keep their face“ (I'm not sure this is the correct translation). While the
  factual difference between „Your idea is piece of crap“ and „We should improve
  on that one“ is very little, the psychological difference is huge. The first
  one *requires* the author of the idea to defend it even if the author *knows*
  it is not good. The latter one is offer of help the author can accept with
  grace and without losing „social points“. If you corner a wolf, it'll fight,
  even when it would otherwise prefer not to.

* Try to understand their points. People often assume you know specific things
  so they don't mention them. They do a bad job of explaining themselves (I know
  I do). Ask them to explain. Assume good intentions from them ‒ they often work
  for the same team, same company, same goals. They have a different vision,
  different needs. Maybe they misunderstood you and trying to understand them
  will tell you that. Maybe they have a point. And even when it comes to worst
  and you really have conflicting requirements, it's much easier to solve it
  when you both know the other's situation. It also helps when you're the
  „losing“ side (eg. when not all your needs are met) if you at least understand
  why that happened and that your position was considered.

* Be clear about your confidence level. Being confident helps persuade others.
  But the goal is not to persuade others. The goal is to find a good solution.
  People often are confident *and* wrong at the same time just to persuade
  other people. If you just feel something, make sure you state it in a way
  where people don't attach more importance than it deserves and it doesn't lead
  the research in a dead end.

* Try to be civil and polite. There's a big difference between British and
  American English (at least as it is presented here, in central Europe ‒ so
  that presentation might be wrong). Suppose you're on someone's lawn, of course
  by accident. The British might say (calmly) something like „Excuse me, sir,
  but you probably should not be here.“, while in America you're as likely to
  hear „Get off my property!!“ (followed by a click of a gun being loaded).
  While both will probably get the job of you leaving the property done, your
  day will feel much better after the British way (and the owner's day likely as
  well). That's not to say nobody's emotions ever get the better of them, so the
  principle of robustness (be strict about what you produce and relaxed about
  what you receive) is a good one here.

* Don't expect it'll go your way every time or that it'll ever be smooth and
  easy. It might, but it's better to expect less and have a positive surprise.

## Problem statement

So, you're designing something. But first you need a clear idea what it is. This
is where a problem statements is a very valuable tool. The [Rust RFC
process](https://github.com/rust-lang/rfcs/blob/master/0000-template.md) asks to
include a motivation and it is a good requirement. But I prefer problem
statement over a simple motivation. While it is about the same thing, there's a
different [framing](https://en.wikipedia.org/wiki/Framing_effect_(psychology)).

In other words, you should have a very clear idea what *problem* you're solving.
If it is a bug (something crashes, data on the output look plain wrong, …), then
it's an easy one to state as a problem. But if it is a user feature, make sure
to include not only the description of what the feature should do, but what the
user might want to solve by that feature. In other words, not only „The word
processor should be able to save the document.“, but „Users need to preserve
their documents across restarts so they don't have to type them all over again
and again. The word processor should be able to save the document.“ This example
is a bit extreme (it is obvious why the word processor needs to be able to save
documents), but oftentimes what feels obvious to you doesn't cross the minds of
others at all.

Be as concrete as possible, give examples, give demonstrations, include some
gathered data if there are some. Avoid being too broad („The ultimate question
of life, the universe and everything“).

If you think everyone should already know what you talk about, because there
were previous discussions about that thing, at least reference the discussions ‒
people tend to assume that everything they have in front of their eyes is
everything that exists regarding the topic and it doesn't cross their minds they
*could* invest some of their time searching for the previous discussions.

If you yourself don't have that problem, talk to someone who does (the user). If
people complain about UX of your product, grab a random person, put the product
into their hands and write down during which activity they cursed the most. Make
sure you understand what their problem is, write it down and let them
cross-check it. Ask them to show the problem to you, make a screenshot, … But
know it's impossible to fix if everything you know is „It's broken“.

I'd even argue that a good problem statement is the hardest part and also the
most important part of the whole design process. Make sure to write the problem
statement down, not only have a vague idea what it is. It'll give you these
things:

* By formulating the problem, you'll be forced to understand the problem better.
  This is the
  [rubber-duck effect](https://en.wikipedia.org/wiki/Rubber_duck_debugging),
  when the solution becomes obvious after you tried to ask for help and had to
  explain the problem to someone else. Talking to rubber ducks may feel a bit
  silly, but writing it down brings no questions about sanity (but keeping a
  rubber duck on your work desk can't hurt, can it?).

* You'll be less likely to get lost in a web of different solutions and end up
  solving something subtly different in the end (something that wasn't a problem
  to start with), ending up invested a lot of resources only to find out the
  problem didn't go away. It'll give you focus.

* You'll get much more support from other people if they know the problem. Maybe
  they don't have the problem and don't ever understand there could be a problem
  in the first place, unless you tell them exactly what is wrong. If you can
  demonstrate the problem so they can feel the pain of it, all the better.
  People should not be asking why you are presenting a solution. By the time you
  do so, everyone should have accepted there's a problem to solve and understand
  what it is (even if they don't *have* the problem, they still need to
  understand *why* it is a problem). This also opens doors at organizational
  structures. Everyone agrees that more performance is good, but won't invest
  any resources in such an abstract concept. If you show the CEO (or whoever
  else holds the money valves) how incredibly slowly a specific web page loads,
  they're much likely to ask what you would need to fix that.

* It gives others opportunity to amend your solution, or to find better
  alternatives. If they don't understand what you're trying to solve, they have
  only the opportunity to either agree or disagree, which is frustrating both
  for you and them. Without it, the discussion can hardly be constructive and
  factual.

* It provides means to verifying the solution later on. You can check it will
  (or that everyone thinks it will) solve the problem. You'll be able to compare
  costs and benefits of the solution. Yes, we engineers have the unfortunate
  tendency to get fixated on solving the problem *no matter what* and come up
  with a solution that works, but is not adequate to the problem.

* If you don't come up with a solution, you still can present the problem
  statement to someone. Maybe they'll have the needed experience, or the
  inspiration and say „Hey, why, that's easy!“. In such case, you still make it
  possible for them to solve the problem and you get the solution you needed.

* In case you have trouble formulating what the problem is, it may be the case
  there isn't a real problem in the first place and nothing needs to be solved.
  Think about that possibility (and try to find a way to verify the theory).

While you can go back to this point during your next phases (because you
experiment with something and it turns out the problem is slightly different,
deeper, …), you should not skip it. By doing so, you risk going in circles,
finding solution to a non-problem, arguing infinitely about if others like your
solution or not or other undesirable things.

Of course, the size of the problem and the solution also influences how much
effort goes into this phase. Smaller problems deserve less attention.

Oh, and avoid doing this *after* you have the solution. You're as likely to come
up with some problem the solution solves, even if nobody was ever bothered by
it.

## Next issues

I'd like to cover these in some future posts (not necessarily in this order):

* How a good solution looks like.
* How to come up with one.
* How one might evaluate the quality of a solution.
* Something about discussing the solutions and proposals in somewhat effective
  way.
