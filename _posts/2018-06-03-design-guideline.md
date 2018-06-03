# Some personal design guidelines

There were some heated discussions in Rust community as of late. During that
discussions, I argued that some guidelines for RFC authors would improve both on
the results as well as the discussions and I promised to give it a try.

However, as not being an RFC author myself (a suggestion to include
stabilization FCPs in TWiR doesn't really count), it is quite a hard position.
Therefore, I decided to give it a more general look ‒ RFC is just a special case
of something I do very often. Design work. So this post can be seen as an
attempt to put words to what works for me and I'll be glad if it can grow into
some more general guidelines ‒ Rust specific or not. If you want to offer a
helping hand building something (not sure what *exactly* yet), you can contact
me on vorner@vorner.cz, we can then agree on some way of cooperation. There are
guides for [learning
Rust](https://doc.rust-lang.org/book/second-edition/index.html), on writing
[unsafe Rust](https://doc.rust-lang.org/nightly/nomicon/), a [guide for hacking
on the compiler itself](https://rust-lang-nursery.github.io/rustc-guide/). But
having a good design is much more important than knowing design patterns or
syntax tricks. Yet, there's no guide for that (or, no Rust specific one).

As with anything, this writeup is from my personal view. This is a collection of
things I've learned over the Internet, several books and practice ‒ I can't
claim much originality here. What works for me doesn't have to work for you.
And as with any guidelines, it is also important to know when to break them. And
I also often fail to follow them.

Also, there's a lot of non-technical stuff around design. Emotions.
Misunderstandings. Politics. I will not try to tell you how to win politic
fights. I'll try to help how to avoid them. I'll assume your goal is to have a
good solution to a problem, not gaining power in corporate struggles. It's
harder to argue against well-stated facts just based on politics.

## General attitudes

While the techniques described here work well enough if you work alone, they
bring much more benefit when in larger group of people. These allow to stay on
track, minimize needless battering and save time.

So, few general rules to help with that:

* Optimize for reading, not writing. By writing a proposal and presenting it,
  you ask a lot of people to invest their time into reading it, understanding it
  and thinking about it. It saves time overall if you can be clear and to the
  point. And people will invest more of their time into the thinking part if
  they save on the previous two and that's what you want.
* Try to avoid personal attachment to any part of the solution. What you strive
  for is a *good* solution, not necessarily *your* solution. Your solution is a
  starting point. This involves a little bit of cheating subconsciousness, but
  in general, things like writing in plural („our solution should“, instead of
  „my solution should“) helps all sides more to work as a team and together
  instead of against of each other.
* Don't force anyone to defend themselves or their (power) position, if you
  don't have to. There's a big psychological difference between „Your solution
  is piece of crap“ and „We should improve that solution, there are some
  drawbacks to it“. The first one is a personal attack on the author, because
  you lower their abilities in eyes of others and they have to defend the
  solution even if they themselves see there are drawbacks. The latter is an
  offer of help and opens future discussion.
* Try to understand other people's problems. Ask them to explain, if they don't
  make their point clear. Maybe they just misunderstood you, or maybe there's an
  actual problem you didn't see or their needs are different. There may be a
  genuine conflict in the needs, but even then, it's still solved in a much
  easier way if you know what is being balanced against what.

## Problem statement

The first step should be understanding the problem being solved. If you don't
know what you're trying to solve, you'll solve something else. Your solution
won't work. You'll get suboptimal solution and you'll make the situation worse.
Or you'll be running in circles, never being happy about your current idea.

Write down your problem, as clearly as possible. This will give you these
things:

* By formulating the problem, you'll be forced to understand it first. This is
  the rubber-duck effect. Sometimes, when you ask for help, you come up with a
  solution just by describing your problem to someone. You don't really need the
  other person there, but if talking to a real rubber duck feels silly, writing
  it to a paper is good.
* You'll not get derailed during the design process to solving something else,
  because you have your written problem statement in front of your eyes. This
  will give you focus. You'll not be investing your time in finding solutions to
  imagined non-problems (in my experience there's no lack of real problems).
* When presenting your solution to someone else, they'll know your motivation.
  If they find problems with your solution, they can offer ways to fix it (or to
  come up with a different solution) while still satisfying your needs. If you
  present just the solution, they'll ask „Why“ and will not understand your
  solution at all.
* Having a concrete problem at hand, one that people can understand (or even
  better, try the pain points of it themselves) opens doors in organizations.
  Everyone knows more security or performance is a good thing, but nobody wants
  to invest the resources in these abstract things. If you can point to a web
  page that loads ages, or you can demonstrate how someone could figuratively
  walk in through your front door and steal all the corporate know-how, you
  don't get ask „Why?“, but „What do you need to fix that?“
* It'll be easier to evaluate your solution once you come up with it. You can
  verify if it helps. You'll see more clearly if the problem warrants such a
  complex solution ‒ usually, the solution comes with a cost, but we get fixated
  on solving it *no matter what*. When seeing that the problem is not so big,
  after all, you can come up with a decision to let it be.
* Include hard data if you have them. Do you have statistics how often the
  problem happens? Or is there a way to gather them?

I believe clearly formulating the problem, in a way everyone can understand it,
is about 70% of actually solving it. If what you have is *only* the problem
statement, that is OK too. You've helped in solving it more than you know, now
the only needed thing is someone with relevant experience or inspiration to read
your problem statement and say „Sure, that's easy!“

While you can go back during the process, you probably should not proceed
further (explaining the solution, or even working on it) until your audience
understands what needs to get solved and agree it needs to be solved.

Avoid overly broad or generic problem statements. Your problem is not „The
answer to the ultimate question of life, the universe and everything“, don't
state it so. If your product is hard to use, write down explicit scenarios where
it feels hard to use. If you are already used to the product and don't see the
problem, grab a random person, put the product into their hands and write down
the intensity of their cursing and during which activity they cursed the most.
If you can't pin-point a specific problem (not necessarily a single one), then
maybe there's no problem at all.

## Coming up with a solution

There's no sure way to come up with a solution and you'll need an inspiration.
However, there are some questions you can ask yourself that often help:

* Is this problem similar to something I've solved before? Could I copy the
  solution? Should I? Did it work previously, or should it be improved on?
* Did someone else solve something similar?
* Is there a way to cheat the problem somehow? For example, there's a lot of
  cheating in 3D graphics ‒ only the side of object facing the user is rendered,
  there are cheap ways to relief shadows by slightly moved additional texture,
  etc, all just to gain some more performance.
* Could I solve some subset of the problem?
* Could I partition the problem into smaller pieces?
* Could I find a solution that doesn't solve it, but makes it smaller (eg. the
  page still loads slowly, but it's a bit better)? Or something that solves 90%
  of cases where the problem arises (eg. caching).

Some other ways to come up with ideas:

* Take a shower or a walk. Let it rest over the weekend. Your subconsciousness
  will work on it while you sleep, even if you don't concentrate on it.
* Take your problem statement (not necessarily in written form) for lunch or to
  a pub or a gym. Bounce ideas off each other. This'll generate mostly useless
  insane stuff, but it may contain something to improve on.
* Experiment with the things at hand. If you can do a 50%-prototype (eg one that
  works in half of the cases), then you have some idea about the problem and
  what it needs.

## Qualities of a good solution

While not all problems have good solutions, or any solutions at all, it is good
to know how a good solution looks like. Sometimes, using a „bad“ solution is
what needs to be done, because the current time constraints don't allow
implementing the good one, or because there's no good one, though.

* The solution feels obvious. The only question it raises is how it was possible
  nobody came up with it sooner. This is similar to a sculptor that says he
  didn't make the statue, that he only freed it out of the extra stone around.
* People who didn't have the problem don't notice the solution being
  implemented. The best solutions are invisible ‒ like gravity. It is
  everywhere, but in general, you don't think about it or pay explicit attention
  to it.
* It is simple. It should be apparent there's no problem. Not just no apparent
  problem.
* There are no serious voices against it ‒ it doesn't have to make everyone
  happy, but it shouldn't make anyone seriously unhappy.
* The solution doesn't go against subconsciousness, it plays with it. This is
  about how intuitive it is. It is also about naming things. A great example is
  Rust's ownership model. Programmers are not used to it, but once they get used
  to it, it feels perfectly natural. Your subconsciousness understands
  ownership. It understands that if you take a cup and give it to someone else,
  you no longer have it. Not that now you both have it, as in other languages,
  that's kind of complex for the subconsciousness to model.

## Evaluating your solution

You probably should evaluate your own solution before you present it. But you
can also evaluate solutions of others.

Some questions that need answering (and the answers may be included in the
proposal):

* Cost vs. benefit study: Every solution costs something, being it time to
  implement it, users learning how to use that, downsides somewhere else… and
  the medicine should not be worse than the sickness. You may want to scale down
  the solution (even at the cost of part of the problem ‒ the 80:20 principle).
  You may want to pick a less intrusive way (in case of language design, a
  keyword is more costly than a function in standard library, which in turn is
  more costly than function in a 3rd-party library).
* Does it actually solve the problem? This is where your problem statement
  shines.
* Is it possible to do it incrementally? Both to have parts of the benefits
  ready sooner and to have early feedback and be able to change the solution
  based on that if need be.
* Is it possible to perform some experiments without actually committing to it?
  Maybe by doing it in a less costly way with some downsides first.  Or, is
  there a way back if something doesn't work? Is there a plan B?

## Presentation and discussions

If you don't work alone, you have to present it and discuss it with others.

* Try to avoid misunderstandings.
  - Be clear about your intentions.
  - Distinguish between what you are sure of and what you only feel or think.
    Distinguish between phases of completeness of your proposal ‒ is it just a
    beer-brainstorming idea or did you go all the way to make a formal proof of
    correctness and implemented a proof of concept?
* Assume good intentions ‒ most people don't try to be mean on purpose. They
  just have different views. Understand them. Make them understand you.
* Try to avoid emotions and attachment:
  - Make sure not to be hostile to people and avoid insults. As the British vs.
    American English shows, there's a huge difference in presentation. While
    equivalent in the meaning, I'm much more likely to comply with „Excuse me
    sir, but you probably should not be here.“ than „Get off my property!!“ and
    I'm much more likely to be polite back to the first. I believe it is much
    less unpleasant to both sides.
  - Try to bring the whole team into the solution, make it „ours“ instead of
    „mine“.
  - Don't worry about improvements to your solution. It is making it better.
* Try to re-read it few times before presenting it (if it's in a written form).
  Let some time pass between the reads and see if it still sounds like you think
  it should.
* Try not to get offended. It's more likely a misunderstanding than an
  intention. The other side most likely just didn't pay enough attention to what
  they were writing or one of the sides doesn't communicate in the native
  language and there's a cultural difference. If you don't manage and get
  offended, try to cool down a bit first ‒ explain you don't like being
  offended, but don't return the offense.
