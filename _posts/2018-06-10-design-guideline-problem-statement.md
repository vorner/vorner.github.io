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
There's a lot of learning books about Rust:
* [The Rust Book](https://doc.rust-lang.org/book/second-edition/index.html).
* [The Rustonomicon](https://doc.rust-lang.org/nightly/nomicon/).
* [Rustc guide](https://rust-lang-nursery.github.io/rustc-guide/) for hacking on
  the compiler itself.
* A lot of others I forgot to list.

But there's not much on design ‒ on what you do *before* you start writing the
Rust code. That phase is probably more important than understanding some
syntactic tricks.

So I'd like this to grow into some kind of guide (I don't know *exactly* what
kind yet). But I'd like input from other people on this (because others may know
different things and because I still feel this text needs improvements). If you
are interested in helping out, contact me on vorner@vorner.cz ‒ we can talk
about some form this could have and how to go about it.

For now, I have some of my own personal observations what worked for me. It is a
collection of things I learned from the Internet, several books and practice (I
can't claim much originality here). And what works for me doesn't have to work
for you. If you have something that works better ‒ well, see the above
paragraph. Also, as with any guidelines, it is appropriate to go against them
from time to time. Use your own common sense. And it is certainly not a silver
bullet ‒ it might help you improve, but following it isn't guaranteed to produce
a good design on its own.

## General principles and attitudes

Design is usually team work. Even if there's one person dedicated to making the
design, others comment, try to find problems in it or validate it. And other
will be implementing it.

Therefore, there's a lot of non-technical stuff around design. Emotions.
Misunderstandings. Politics. Usually, I try to avoid them as much as possible,
but they are still there, we are still human. My goals are usually to have a
good solution to a problem, not gaining points in corporate power struggles. So,
good design is as much about the technical aspects as about psychology,
negotiations and keeping one's sanity.

There are usually people that argue hard. I know I do, sometimes. It's not
because I'd like arguing, but sometimes I feel there's a need to do that. But if
there are good factual arguments at table, if all the parties try to communicate
in a clear way, it goes smoother. And remember, these disagreements are
professional, not personal ‒ you still can go to lunch and have a good chat
about cars (or whatever else) even if you disagree with each other.

So here are some little overall tips to lead a good discussion:

* Optimize for reading and understanding, not for writing. Whenever you write a
  design document, or comment on it, it'll be read by many more people. It saves
  time many times over for them if you are clear about what you wrote. And it
  saves *your* time if you are not misunderstood and you don't have to explain
  again.

* Try to avoid personal attachment to your code or ideas. The solution belongs
  to the team. It helps cheating your own subconsciousness ‒ if you talk about
  „the idea“ instead of „your idea“, it's much easier to have it replaced by a
  better one.

* Avoid forcing people into the defensive. If you corner somebody by saying
  „your idea is piece of crap“, you gave them no option but to defend it even if
  they see it's not good. If you suggest *the* idea could be improved on, they
  can allow the solution to be changed completely with grace and without losing
  any social points.

* Try to understand their points. People often assume you know specific things
  so they don't mention them. They do a bad job of explaining themselves (I know
  I do). Ask them to explain. Assume good intentions on their part. They likely
  work for the same team, same company, same goals. They have a different
  vision, different needs. Maybe they misunderstood you and trying to understand
  them will tell you that. Maybe they have a point. And even when it comes to
  worst and you really have conflicting requirements, it's much easier to solve
  it when you both know the other's situation. It is also easier to give up some
  of your own requirements when you know the reasons for it and that your
  position was considered.

* Be clear about your confidence level. Being confident helps persuade others.
  But the goal is not to persuade others. The goal is to find a good solution.
  People often are confident *and* wrong at the same time just to persuade
  other people. If you just have a gut feeling, make sure you state it in a way
  where people don't attach more importance than it deserves and it doesn't
  mislead the research in a dead end (but don't disregard gut feelings either).

* Try to be civil and polite. Personal attacks distract the attention from the
  facts to meaningless fighting. If you disagree with someone, you need to state
  that, but they should see the reasons why you don't agree, with cold head, not
  think about violence when remember your contribution to discussion. But also
  remember emotions get the better of everyone from time to time.

* Don't expect it'll go the way you planned. If you propose a solution, it'll
  likely be changed after a discussion ‒ usually to the better. If it doesn't,
  it may mean the idea was good to start with, but it also could mean nobody
  cared to read it.

## Problem statement

So, you're designing something. But first you need a clear idea what it is. This
is where a problem statements is a very valuable tool. The [Rust RFC
process](https://github.com/rust-lang/rfcs/blob/master/0000-template.md) asks to
include a motivation and it is a good requirement. But I prefer problem
statement over a simple motivation. While it is about the same thing, there's a
different [framing](https://en.wikipedia.org/wiki/Framing_effect_(psychology)).

In other words, you should have a very clear idea what *problem* you're solving.
Bugs are easy to state as problems. But even features can be described as
problem statement (or user story). So, don't say the word processor needs to be
able to save documents. Say that if it can't users have to type their documents
every time from scratch. This is a bit to the extreme ‒ we all *know* why word
processors can save documents. But oftentimes, what seems obvious to you
(because you spent some amount thinking about the problem) doesn't even cross
the minds of others.

This is not to say you can't do anything just for the fun of it, but you should
know it.

As for the form, be as specific as possible, give examples, show demonstrations.
If you think everyone should already know what you talk about, because there
were previous discussions about that thing, at least reference the discussions.
If you don't, they'll assume what you gave them is everything relevant (it's
called WISIATI by psychologists).

If you yourself don't have that problem, talk to someone who does (the user). If
people complain about UX of your product, grab a random person, put the product
into their hands and write down during which activity they cursed the most. Make
sure you understand what their problem is, write it down and let them
cross-check it. Ask them to show the problem to you, make a screenshot, … But
know it's impossible to fix if everything you know is „It's broken“. A good
problem statement and a good bug report share many common qualities.

I'd even argue that a good problem statement is the hardest part and also the
most important part of the whole design process. Make sure to write the problem
statement down, not only have a vague idea what it is. It'll give you these
benefits:

* By formulating the problem, you'll be forced to understand the problem better.
  This is the
  [rubber-duck effect](https://en.wikipedia.org/wiki/Rubber_duck_debugging),
  when the solution becomes obvious after you tried to ask for help and had to
  explain the problem to someone else. Talking to rubber ducks may feel a bit
  silly, but writing it down brings no questions about your sanity (but keeping
  a rubber duck on your work desk can't hurt either, can it?).

* You'll be less likely to get lost in a web of different solutions and end up
  solving something subtly different in the end (something that wasn't a problem
  to start with), ending up having invested a lot of resources only to find out
  the original problem didn't go away. It'll give you focus.

* You'll get much more support from other people if they know the problem. Maybe
  they don't have the problem and don't ever understand there could be a problem
  in the first place, unless you tell them exactly what is wrong. If you can
  demonstrate the problem so they can feel the pain of it, all the better.
  People should not be asking *why* you are presenting a solution. By the time you
  do so, everyone should have accepted there's a problem to solve and understand
  what it is. This also opens doors at organizational structures.

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
  there isn't a real problem and nothing needs to be solved. Think about that
  possibility (and try to find a way to verify or disprove the theory).

While you can go back to this point during your next phases (because you
experiment with something and it turns out the problem is slightly different,
deeper, …), you should not skip it. By doing so, you risk going in circles,
finding solution to a non-problem, arguing infinitely if others like your
solution or not or other undesirable things.

Of course, the size of the problem and the solution also influences how much
effort goes into this phase. Smaller problems deserve less attention.

Oh, and avoid doing this *after* you have the solution. You're as likely to come
up with *some* problem the solution solves, even if nobody was ever bothered by
it.

## Next issues

I'd like to cover these in some future posts (not necessarily in this order):

* How a good solution looks like.
* How to come up with one.
* How one might evaluate the quality of a solution.
* Something about discussing the solutions and proposals in somewhat effective
  way.
