# Should you learn Rust?

Oftentimes, I see a variant of this question posted or asked somewhere. In
general, most of the times I think the answer is „Yes“, but maybe for reasons
other than you'd think at first.

First, the trivial things out of the way. If you ask me if you should learn
something, most of the times I'd answer yes simply because learning *anything*
is good for you. It broadens horizons, stimulates the brain and makes you step
out of your comfort zone. Besides, you never know what knowledge or skill comes
handy. And Rust definitely is something stimulating to learn and you'll step out
of your comfort zone doing so.

But if presented with the question „Should I learn Rust *or* this other
language?“ (because everyone's capacity to learn and time is limited), then it
needs some explaining why I'd think it still is a good idea to choose Rust,
especially if you already know some other language (Rust *might* be a bit hard
as the first language).

If you consider Rust because you already know C or C++, find it annoying to work
with or slow to code in, then you probably already have your answer. If you know
you want the challenge of learning Rust or you find the community nice, then
this as good reason as any. If you want a good, enjoyable free-time language,
then go ahead.

But if you ask if Rust will help you get a better paid job, then it is a little
more complex. Getting a Rust job is not easy (it's probably easier to convert
an existing job to a Rust one, but that takes a lot of social skill and a lot of
time). So how could learning Rust help you to a better paid job? I think it
could. And not 5 years from now (when companies start hiring Rust programmers),
but now.

Let me digress here a bit. There are programs of different qualities. Some of
them are mostly bug free. Someone wrote them, put them to production and forgot
about them for two years, until the user base grew hundredfolds and it needed to
be extended. On the other hand other programs bring attention to their existence
on weekly basis, with a fresh bug every time. You can guess which ones I like
more (both to have around an do be writing). You can guess which ones the
companies prefer to have running on their servers (even though some of them fall
pray to the illusion the other ones are cheaper to make).

What sets them apart? Is it the tool used to build them? Is it some trick, dark
magic or technique, transferred under the promise of silence from master to
novice? Praying to the right god before each commit?

While some techniques or tools exhibit a tendency to help or hurt the quality of
the code, these are just statistical tendencies. I'm pretty sure I could find
an artful masterpiece in whatever language, as well as terrible piece of crap
(yes, that also includes Rust). I could find failed projects despite them
following every piece of good advice out there.

However, I know people who reliably without any exception produce the
masterpieces, no matter how inadequate tools you arm them with, as well as
people who produce substandard quality results despite all the tools you give
them. It is the first kind that chooses their employers at will.

So if it is all about people, why does the industry strive to produce the better
tools? Mostly, because it is easier to produce good-quality tools than
good-quality people, but for other reasons as well.

How do the bugs come into existence in the first place anyway?

A bug is a consequence of a broken, confused or completely missing mental model
of the problem and program. It is not the programs that are being confused. It
is *us*, their authors (yes, I know, that is a bit humbling realization,
considering all the broken software out there). To create a good program, one
has to first attain a clear and correct understanding of the problem, the
solution, to know it inside out. And there are many challenges to this. The most
obvious one is the fuzzy human brain, but unclear requirements, the sheer size
of the program, its age, the team (there are `n+1` versions of the vision in a
team of `n` people) or companies *not* wanting their employees to think also add
to it.

To get a good program, also some technical good practices are needed ‒ like
paying attention, having the time to do it, etc. But bugs caused by this are
often easier to fix than the ones coming from the broken vision.

## Things that help

This is where our tools come into play. They help our limited brain power in the
fight with the many-headed hydra of confusion. However, many of them need to be
applied with care and moderation or they could hurt more than help.

* Splitting the problem into separate, mostly independent pieces helps, as one
  needs to keep only the model of the one smaller part in mind at each given
  time. The big problem can be split up between people or teams. If not split,
  one gets a spaghetti mess of code (everything is interconnected and it's
  impossible to make sense of it). The other extreme is Lasagne code ‒ if
  partitioned into too many too small pieces, when there are just too many
  layers and it's impossible to find the place where something actually
  *happens*.
* Writing (drawing, imagining, …) a design up-front, even though it isn't
  followed 100% in the end. It helps to have a vision instead of cancer-grown
  code that works only because of luck. This helps fighting complete *lack* of
  the mental model, as well as helps solidifying it to some extent ‒ in other
  words, think before starting to debug an empty program.
* Reasonable amount of tests (somewhere in between of none at all and boring
  everyone to death with testing trivial getter methods). These don't only help
  you catch a regression right after it is introduced to the old code, but it
  also helps you check your assumptions about the code and state how the code
  should be used (which is a valuable archaeological artifact with legacy code ‒
  tests tend to be less out of date than documentation). As a note, I often
  find that it makes sense to write some tests before the actual code (as TTD
  states) to help me understand what result I want to achieve, and some after,
  when I know where the tricky parts of code lie and aim the tests at the most
  vulnerable places, to make sure I didn't break these parts.
* Code reviews help to catch bugs made out of not paying attention. More
  importantly, they help to unify the team mental model (it's still `n+1`
  different versions, but the differences are smaller) and invite challenging
  the mental models of each other.

## Languages

So, how does this relate to Rust or languages in general? I noticed few things:

There are languages in which I produce mostly bug-free code. This is code I
wrote and after trying it once or twice, fixing few trivial things, I'm
reasonably confident it does what it should (and it later on turns out to be
mostly true). And there are languages that often lead to code I'm just not
confident about all (or I'm confident the code is broken, but don't want to
spend the time fixing it).

The first group contains Rust for me. It is not a big surprise, as it is
language specifically *designed* to help write correct code. But I think the
advertised reasons (preventing segfaults, data races…) are the less important
part (they are still very valuable, as they prevent the most dangerous
catastrophes, but it's not what helps against most of the bugs, or the most
frustrating ones). What might be a surprise for many is that this first group
also contains Perl for me.

On the other hand, if I want a reliable code I'd never willingly choose Python
(note that this may differ for you, this is nothing against Python, and even I
choose it for some tasks ‒ just not the ones where reliability is the key
factor).

However, I also noticed that after learning Rust, the code in *all* the
languages I write in improved. Why?

It comes back to the mental model of the problem. Rust has a very strict and
tight type system. If your mental model is confused, something will not align
in it properly and it won't compile. It doesn't matter if it's lifetimes, types,
or just getting a *very* hairy interface (which you, in the end, beat into
submission, but feel that it's not supposed to be that way), you'll get a
feedback that what you wrote is wrong in some way. It'll kindly remind you to
step back and think about what you try to do. Haskell is very similar in this,
but it also kind of assumes you are an expert in algebraic theories ‒ so if you
like algebra and formalisms, you might get the same benefits from Haskell, and
you may enjoy it more than Rust (or you may find that you need neither of these
two, because your thinking is precise enough already and there's nothing more
they can teach you).

Furthermore, Rust's community has a large concentration of the kind of
programmers that produce the masterpieces. If you hang around, you're bound to
learn something from them ‒ curiosity and good thinking habits *are* contagious.

With Perl ‒ at least for me ‒ it's better expressiveness of the languages. Perl
was made by a linguist and whatever mental model I have is easier to transfer to
it than to other languages (for example, while technically equivalent, there's a
slight difference in the connotation between `if not` and `unless` to the human
brain). That mental model can still be wrong somehow, but at least it doesn't
get worse by the translation to the language and I can think with less
restrictions, closer to how my brain works, not how the language works. Also,
the fact Perl programs are just terrible to debug and fix makes me pay much more
attention to the thinking upfront as well as the implementation. If Rust is
martial arts teacher, Perl is a pub brawler. If you survive either, you're
likely to be good at defending yourself, though both can be painful at times.

Taking Python, the language tries to be *comfortable*. Which isn't a bad thing
in itself, but it lets one *slack* in the thinking. Where Rust's type system
breaks and shatters on the wrong mental model, Python's simply bends. This is a
good thing when the goal is to get something done quickly and the quality
doesn't matter that much (learning the first steps, writing tests for other
software, prototypes, one-time analyses of data, something that runs on premises
and under surveillance of the author), as it doesn't force one to make the
proper mental model. It just kind of works, without that much effort. Imagine a
donkey in medieval time ‒ traveling on a donkey wasn't especially elegant or
fast, or fancy but it was reasonably cheap and got you from place A to place B,
probably without throwing you off and breaking your neck. It just didn't make
you exceptionally good at anything, while riding a horse (especially a wild one)
could.

## Conclusion

Even if you don't expect to write a single line of Rust code for money in
the next two years (I hope companies learn by then that Rust is a good solution
for some of their needs), learning Rust is very likely to help you get a better
job by becoming a better programmer overall. There are easier languages to
learn, but it is because learning Rust is 10% learning its syntax and rules and
90% learning how to think properly. Most of that 90% is applicable in other
languages, they just don't mandate that you do.

In other words, Rust has your back when it comes to making sure you thought
about the problem and that you have solid model. In other languages you're on
your own, but at least you know what you aim for.

Therefore, you probably should learn something like Python first (if you haven't
already). It'll let you get away with almost everything. This is good as the
first step, to get familiar with programming in general, what it is about (and
then keep it in your toolbox for times when it is the right tool for the job,
same as every DIY person has a lot of duct tape around). Then something that is
a good teacher, which could be Rust. As it is said, smooth sea never made a good
sailor. But you better start your first attempt at the smooth sea.

And then, lot and lot of practice. Of course you don't reach mastery just by
learning a language or two, but by spending thousands of hours fixing errors
(made by you and others) and learning from them.

However, one warning. You'll have easier time getting paid more, but also harder
time picking an employer that can give you satisfying problems to solve.
