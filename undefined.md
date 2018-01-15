# About the undefined behavior

One hears a lot about the dreaded Undefined Behavior in programming. It's a big
topic for Rust, because it tries to solve the problem. And there are many
definitions about what it is. However, I feel some aspects of it are not
appreciated enough. Therefore, this is what I think everyone (well, everyone
close enough to computers) should know about it. Like the stories about why kids
shouldn't talk to strangers, this fairy tale should be told to kid-programmers
before bed time.

The term was coined in the C standard, and most other languages just plain
borrow the term. The standard actually talks about several „lesser“ levels of
undefinedness:

* Defined behaviour ‒ all is well and nice. `1 + 1` is `2`. Always.
* Implementation defined ‒ some things are defined not by the standard, but a
  concrete implementation (combination of the target machine, compiler,
  ...). Things like the size of a pointer is different between platforms, but
  stays the same through the program.
* Unspecified ‒ it still acts somehow sane. For example, the value of „ordinary“
  global variables inside a signal handler is unspecified. But the variable
  still has *a* value ‒ reading an `int` global variable there is safe in the
  manner that the program won't explode while doing so.

*Note: the previous version of the text stated that one such unspecified
behavior is `++ x + ++ x`. That is technically correct, but this is **also**
undefined behavior (by definition, every undefined behavior is also
unspecified).*

## Undefined behavior

This is basically „all bets are off“. You can't have any expectations whatsoever
about what happens if it is invoked. Monsters with tentacles from other
dimensions and such 🐙.

This usually happens because the standard (or something else) says some
situation can never happen, but this assumption is broken by the code. Like,
integers never live on odd addresses (since there's actually no *legal* way to
put them there in the first place, so it makes no sense to prescribe what
happens if one gets there nevertheless). Or dereferencing an invalid pointer.
Indexing after the array's end. *Creating* (not even using) a pointer further
than 1 element past an array's end.

Let's see a bit of code:

```C
char c[4];
memcpy(c, "hello", strlen("hello"));
printf("%s"\n, c);
```

If this is ever executed (don't do that, you might regret it!), it can do any
of these, but also whatever else:

* Not compile.
* Print "hello", followed by the complete works of Shakespeare. You know,
  there's no terminating 0 to the string and a book reading software resided in
  that part of memory before this program got executed.
* Crash in a nice way with a segmentation fault when copying to `c`.
* Crash in a nice way with a segmentation fault when printing "hello".
* Crash in a nice way with a floating point exception some time later on.
* Print "hello" and then format your hard drive `h`. You noticed that `o`
  doesn't fit into the array, didn't you? Well, it could have overwritten the
  return address of the current function, to an address of a function in the
  standard library for formatting hard drives (why do they even put these in
  there?). Bad things happen.
* Crash in a very rude way after uploading the complete works of Shakespeare to
  Facebook, making you liable for copyright infringement.
* Print "hello".

And all this just because `printf` assumes that it gets a valid string that
fits into the memory allocated for it. What we did instead was the programmer
equivalent of putting a cat into a microwave ‒ it was obviously never meant to
be used that way and nobody thought about what might happen if you do such a
stupid thing, because, well, it's just plain stupid to do. Except when the
situation is really complex and you pay little attention to what you do, you
do such a thing by an accident.

You noticed I put the „correct“ behavior last. This is on purpose. They are
ordered by the perceived severity.

This is what I'm trying to get to from the beginning. The absolutely *worst*
thing about undefined behavior is that it is allowed to *pretent to work as
expected*.

The issue may fly under the radar a very long time ‒ passing tests, getting into
production and then making the autopilot crash the plane. The very fact that the
code is linked with tests or not may change its behavior in unpredictable ways.

Therefore, you don't want any undefined behavior in your code. Ever. There's no
kind of „safe“ undefined behavior. And the fact that your code doesn't crash is
not a good enough indicator that it does not contain it. If someone tells you
„you know, what you do is wrong, this is actually undefined“, the right answer
isn't „but it works OK“. It works OK now, but it won't tomorrow or the day
after.

## Downgrading undefinedness

Of course, a C compiler (or whatever compiler) is always allowed to downgrade
the undefinedness of some case. It may promise to always execute the
pre-increments from left to right, for example, and make it from unspecified to
implementation defined. It may promise that whenever you do the `hello` thing
above, it will always print the complete works of Shakespeare and then crash in
very rude way, insulting your ancestors. It may promise that whenever you try to
use an integer on even address, the program will crash in a well defined
specific way.

## Rust

Where does Rust stand in all this? Well, it tries to tell you „you know, what
you do *looks* wrong“ before you ever try running the code („why are you
carrying that cat to the microwave?!“). It may actually be OK, but Rust tries
to err on the safe side (pun intended). You can still tell it „get of my way, I
know what I'm doing“. But you should do so only if you *really* know what you're
doing, not because the compiler annoys you.

I might be paranoid, but whenever you write `unsafe` into your Rust code, you
should make a proof (well, not necessary a *formal* proof, but at least a
reasoning) why this use of `unsafe` is actually safe to do. And put it into a
comment there, so others can see your reasoning. In addition, you should also
state the reason why the `unsafe` is necessary in the first place ‒ simply
because we all do mistakes in our proofs, formal or not, and why risk it.

In other words, it is great to have a language that kills at least the last and
worst kind of undefinedness (note that it still has the others!) and if it
crashes the plane, it does in a well defined and repeatable way. You can
actually test for that.

## Other languages

Some languages are safer than others. There's however nothing such thing as a
safe language ‒ you still can crash the plane even when staying in the realms of
defined behavior. There's still no replacement for using your brain.

Also note that not all undefined behaviors are caused by pointers, as would some
proponents of so-called safe languages make you believe. For example, a data
race (multi-threaded access to data without synchronisation while it is being
modified) is undefined *on the hardware level* (well, it's unspecified from the
hardware point of view, but that can lead to having arbitrarily wrong data in
arbitrary places, which makes it undefined higher up in software). Unless the
language does something about it explicitly (eg. by making sure only one thread
runs at a time, like in Python, or that you never can change a thing after you
assigned it, like in Haskell), it is possible to invoke it.
