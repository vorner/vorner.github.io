# Why is Rust difficult?

Rust is considered difficult to learn by many people. Indeed, when I learned it,
I considered it to be the hardest programming language up to that time I've met.
And that says something, since I have a habit of learning whatever language I
find interesting. Looking back, I'm not sure I was correct, C++ is probably
harder ‒ but it was distributed into much longer time than learning Rust.

So, this is my point of view what I believe are the reasons. Also, I'll claim
this upfront ‒ I don't consider it necessarily a bad thing for a language to be
hard to learn, not when I get something in return for the investment.

## Rust attacks hard problems

There are languages that are simple. To point one of them, look at
[lua](https://www.lua.org/). The complete language manual is shorter than
introductory book to most other languages. It has something like 4 or 5 data
types and you can't add your own. Why is it so?

Because lua, and other languages, made different choices about what their goals
are. If you start designing a language with the intention of creating a very
simple and minimal language, you will end up with a very simple and minimal
language (if you're any good at designing languages). The language may not be
suitable in some situations, but that wasn't the goal.

On the other hand, if you declare upfront that your language needs to be able to
solve any hard problem anyone thinks of, run fast and be safe to use then you'll
probably get your wish, but it'll hardly be simple.

And that's what the authors of Rust decided. To be able to do whatever is needed
done. But Rust is not the only complex language because of that. Look at C++ ‒
Rust at least keeps the complexity somewhat sane.

However, this is difficulty that anyone trying to enter the low-level world of
system programming should expect. All these details about where a memory is
allocated, how much memory it takes, that were not important in other languages
suddenly surface. But that's because you need the control, isn't it? Otherwise,
you *would* use the simple language, as it would be sufficient.

## The language is honest

This is similar to the above, but not completely.

When the language designers want to solve a problem, there are two general
approaches.

First one is to hide the problem from sight and let the language handle it
somehow. The fact it isn't always the best way doesn't matter.

The other approach is being honest about the complexity of the problem, admit
it, but arm the programmer with tools to handle it.

This means the beginner must grasp the complete complexity of the problem, but
makes it easier in the long term ‒ simply because after solving the problems,
there's no beginner any more and because sometimes having the needed tools and
not fighting the language to get to the problem is just easier.

## Rust is different

One of the problems why I found Rust hard to learn was that it looked similar to
other imperative languages on one side, but introduced a lot of novel concepts.
It has all these cycles, conditions, functions, like everyone else. But it also
has the ownerships and traits and lifetimes. These do have its reasons, but they
feel alien at first ‒ but not alien enough to completely switch mental models.
But that also make the language interesting.

What I mean about switching mental models. If you coded in Python and then tried
Haskell, you'd see at the first glance than *nothing* is the same. But with
Rust, I started to code like I did in C++, because it looked a bit like C++. Put
some callbacks here and there… and then the compiler complained, because such
model doesn't fit. I tried to force it and after a time I got a really hairy
API I didn't like a bit.

The problem here is two-fold:

* It is not clear upfront the language is different than whatever else one might
  already know and that the other knowledge needs to be put on a shelf for a
  while.
* There are many books about good design in the OOP world, many books and
  classes how to structure a program in pure functional world. But not many
  (none?) books about best practices for a world based on ownership and
  composition.

## The compiler is a very strict teacher

All things together, Rust insists that your program will be correct or it won't
compile. Strict typing makes you think about the relations in your program. It
checks that you don't get data races. It will tell you if you try to free some
memory too soon.

This is a good thing. It makes sure your programs are better than of the
competition (if you manage to compile them, of course), that they are less
likely to crash and burn or something even worse. It makes the language easier
to use (imagine if you had to actively *think* about *all* these things instead
of relying on the compiler to check it for you ‒ welcome to C).

But at the same time, it makes learning it a bit harder, because it insists on
you learning everything needed to write a good program. An average is not
acceptable.

If you just learned about how to write a function, it doesn't feel very
encouraging when the compiler starts to talk about data races (what are these,
anyway, you ask at the time). A program that is subtly wrong would be fine at
that time ‒ at least it would run.

But in the long run this makes you a much better programmer. Even in other
languages.

Today, there's a lot of very good documentation about this ‒ it at least
explains why such things are incorrect and how to fix them.

## People say it's hard

This one comes down to perception. If so many people say Rust is hard to learn,
then it must be true. The fact they learned in times when there was no or little
documentation doesn't change a thing here.

## What to do about it

Documentation and resources help a long way. However, I believe the biggest
issue is perception and expectations.

So, if someone asks (or if you're considering learning Rust and ask yourself):
yes, Rust is hard to learn. But not necessarily *slow* to learn. A month or so
of experimenting in the evenings after work goes a long way. By that time, you
probably could start contributing to the Rust compiler itself, with some little
help ‒ and compilers are one of the most complex pieces of software that exist.

What should a beginner in Rust expect:

* Some frustration that his or her programs don't compile on the first (or
  tenth) attempt.
* Having to understand a lot of problems not known to exist before.
* Satisfaction after discovering how much can be learned with a very strict
  teacher and probably enjoyment from the learning itself.
* Find the language different and interesting.
* Becoming a better programmer, even in other languages (for example, ownership
  is just formalisation what most people do in languages with manual memory
  management anyway, but Rust makes sure it becomes second nature).
* Meeting some very nice people in the community.
* Discovering that it's possible to both write and run programs fast at the same
  time instead of having to choose.
