# The dark side of ergonomics

**Disclaimer**: The topic I'm going to write about is somewhat controversial and
potentially unpopular. My intention is not to troll or provoke flame wars or to
hurt anybody's feelings. Don't let my disagreement with something discourage
you. If you and people like you didn't do such an overall great job with Rust, I
wouldn't bother disagreeing about anything. My intention here is to share a
different point of view and to initiate a legitimate discussion, not an attack.
Therefore I'll ask for something. Disagree with what I write about if you wish,
but try to consider it. And if you have a strong urge to comment, do comment,
but maybe try to give the feeling a 30 minutes to cool down. I have these
feelings too and I promise I'll try to do the same ☺ (I've been re-reading this
very article for the past few hours already).

Despite having an experience with wide range of computer languages, including
C++ and Haskell (both strong influences to Rusts design), I found Rust hard to
learn. Sometimes I grind my teeth about something the compiler doesn't let me
do. Despite that, I didn't put ergonomics as a wish in any poll. In fact, if I
was to take a poll right now, I'd probably be *against* further ergonomics
initiatives. Rust never did anything seriously *evil* to me, which I probably
can't say of any other language I've used on a real project. It has it's
never-ending stream of paper cuts, but they stay just that.

Maybe you think I'm crazy. Why would anyone *not* want their language easier to
learn and use? Maybe you think I'm an old grumpy who is against all the new
things out there to have. Maybe you'd be partially right about both of these.
Maybe I just have dealt with more bugs than I like and have seen far too many
things broken and I'm over-paranoid. That's what years of C++ (and other
programming, to lesser extent) do to you. After all the time juggling chainsaws,
I'm afraid to change anything that is provably *not* a chainsaw, in a fear it
might turn into one.

But apart from these, I think ergonomics is an double-edged sword. I see another
Rust goals more important. It's its robustness, or confidence or zero-bug
abstractions, or how to call it. Simply the fact that if my code compiles,
there's a very good chance it is correct and that if I put it into production
today (without further testing), I'll find it running there tomorrow. That is
quite unique, at least between mainstream-ish languages.

On a very high level, ergonomics and this robustness necessarily go against each
other. I reach robustness by refusing broken and *suspicious* programs at
compile time. I reach more ergonomics by accepting more programs and *hoping*
they do what the author have meant, so I don't have to bother the author with a
compilation error.

So, while I know a lot of people *want* more ergonomics, I'm not sure it is what
they [*need*](https://en.wikipedia.org/wiki/Nanny_Ogg#Role_and_power).

This is not only a theory. Let's do few case studies. These are kind of extreme
cases that better illustrate the idea. These are *obvious* cases where this
exact ergonomic improvement would hurt the overall experience.

I don't claim I've seen any serious proposals to change exactly this. But
subtler things exist, and it's when *I* get the strong urge to comment on RFCs.
Not to discourage the authors, but because I have a different point of view.
Both of us want the best for Rust, we just have a different opinion on what it
is.

## Empty values

Everyone knows about NULL-access bugs or their language-specific variants (`None
type has no attribute` in python, `NullPointerException` in Java, etc). Let's
not talk about the seriousness of the consequences. In whatever language,
accessing such empty value is a bug. OK, there are languages where you get some
kind of default value (0 if you expect a number, an empty function if you call
an empty value), but that's *likely* a bug too. If you rely on that and do it on
purpose, better place a comment there.

Rust has its own variant (unwrap on `None` panic). How comes it happens less
often in Rust than in Java? Because in Rust, the action to access something that
might be `None` is *explicit*. Typing of the `.unwrap()` makes me acknowledge it
*could* be `None`. You know, the effect „Oh crap, I better handle that too,
right?“. It forces me to fix my *mental model*, and not write the bug, instead
of fixing *the bug* later on.

But would automatic coercion of `Option<T>` into `T` where needed (some kind of
auto-unwrap that would panic if it is `None`) be more ergonomic? Yes, it
probably would. Who wants to put all these `unwrap`s and `match`es in their
code?

Every time I have to handle possibly empty value or an error, I remember how
many times that saved me a multiple-hour bug hunts on systems I don't have
direct access to instead of thinking how that is so *unergonomic*.

## Implicit flow control

Yes, this is about exceptions. In many cases they are more comfortable than
having to handle it on every level. You just don't have to care about what and
if it might go wrong, right?

Wrong. Especially if you have a language without a garbage collector and you
don't consider exceptions to be fatal or almost-fatal (basically making them
into Rust's `panic`), you have a *very* hard time. And this hard time has a
name. It's [Exception safety](https://en.wikipedia.org/wiki/Exception_safety).

You see, the fact that you don't have to explicitly handle errors is outweighed
by the fact that you have to structure your code in a way every point in it
might throw an exception at you. Basically, not caring *what* may go wrong means
you have to assume that *everything* might. It means you either have to do
complex rollbacks in RAII guards (or `finally` blocks in other languages), or
have all your state consistent at all times. Even if you want to provide a weak
exception safety, you have to make sure that at least your destructor won't blow
up as a result of the mess of a state you left when the exception flew by. I
don't even talk about the magic *the compiler* needs to do behind the scenes to
make the RAII guards work.

And while C++ has the `noexcept` keyword, that can annotate a function or method,
it is virtually useless. It doesn't check at compile time the function doesn't
throw. It just turns every exception that would be thrown into an abort of the
program at runtime. Furthermore, there's no indication of except/noexcept
semantics on the caller side and removing the keyword from the function's
signature won't make all the places that rely on the noexcept semantics not
compile.

Compare it with Rust and it's `?` operator. Yes, you have to explicitly
propagate all „exceptions“ upwards. Manually. That's a gruelling work. And you
have to actually *think* about doing that. But when you add or remove
fallibility from your function, the compiler will point at all the places where
the change matters.

If anyone ever proposes to have a context where they propagate automatically,
I'll be in the first line, trying to explain it is not a good idea. Yes, I've
worked on a project that mandated strong exception guarantee everywhere, even on
`std::bad_alloc` exceptions. I don't want more of that. I want to see the places
that can „throw“ and the places that are „throw-safe“. And not only in code I
wrote, but in all the code I have to read when something goes „Ooops… this was
never supposed to happen“.

## Automatic type conversions

Automatic type conversions make programmers' lives easier by not forcing them to
write type-casting, right? No stupid `.into()` anywhere. Or so the C++ committee
thought so long ago (it introduced the `explicit` keyword since then ‒ funny how
many things can be „solved“ by adding yet another keyword nobody learns to use).

This one is quite recent experience for me. This prolonged one of my bug-hunts
by about 3 hours, because it led me down the wrong rabbit hole.

Let's say you have this class. It came from 3rd party library, so you don't
really know it, but after desperate whole-day battle with the stupid piece of
program, you instrument the library with debug-prints in addition to your own
code. You know, because *nothing* besides the debug prints works ‒ the code
crashes at a random time after running for few minutes at full speed (stepping
through it in debugger would take forever), the crash manifests long after the
corruption causing it happens, it doesn't happen in valgrind at all (which hints
at some kind of race condition), [`rr`](http://rr-project.org) gives up on you
because of some oddball syscall nobody but you ever thought of using *this exact
way*, trying to prove your code correct only filled heaps of papers with notes
without success and after all you're *grateful* that you have at least the debug
prints and can then grep through the multi-gigabyte log file. Then add more
prints, let it run for few minutes, rinse, wash and repeat. At least debug
prints are reliable, if nothing else. After all, all is good, because the
program crashes reliably on your own computer, that basically means its down to
having more patience than the bug.

```cpp
class Token {
private:
	uint32_t value;
public:
	// Some fields omitted
	operator bool() const {
		return value;
	}
	bool operator ==(const Token &other) const {
		return value == other.value;
	}
	bool operator !=(const Token &other) const {
		return !(*this == other);
	}
};
```

(edited to fix trivial compilation errors)

What do you think this bit of code will do?

```cpp
const Token tok1 = getUniqueToken(), tok2 = getUniqueToken();
if (tok1 != tok2) {
	std::cout << tok1 << "!=" << tok2 << std::endl;
}
```

If you correctly guessed that it'd print `1!=1`, then consider yourself a member
of the „I've seen more broken code then I care to count“ club. And that's when
all the relevant code is together. It wasn't for me, and it wasn't exactly this
code, so I went hunting why the token was always 1 ‒ which it wasn't. And if you
didn't guess, consider yourself lucky and here's the hint: there's the `operator
bool` (autoconversion to `bool`) and *no* `operator <<` for this type.

Anyway, this ergonomic *improvement* broke even the all-reliable debug prints
for me at the time when I was least likely to appreciate the joke.  I don't
consider the C++ committee to be incompetent people. On the contrary.  And even
them couldn't foresee all the consequences, so let's learn from the history.

## The actual message

As a conclusion, I'd like to say that I'm not against ergonomics per se. I'm
just a bit afraid of it, because I've seen too many ergonomic improvements in
different languages to backfire, either separately or when interacting with
other language features.  I'm not claiming C++ (which dominates two of the above
examples) is in *any* way ergonomic, I'm claiming these features were introduced
in the *name of ergonomics* and I reached for it mostly as a good source of
these (since I use mostly C++ lately, Rust unfortunately coming second in line).

So whenever I see new ergonomic proposal, I'm a bit sceptical. Rust works
*great* compared to the above. And, you know, don't fix what ain't broken.

I'm probably too afraid of disrupting the *quite* good and unique properties of
Rust. It's not that I'd be against everything and anything.  For example, `NLL`
is great in my opinion, because it passes a „litmus test“ ‒ even when trying
hard, I didn't come with a scenario where it could mask a bug.

I fully understand that my balance of what is worth and what isn't is severely
skewed by living too long in the face of danger from both complex, long-lived
code bases and not very cooperative languages. I probably see monsters
everywhere, even where they are not. But isn't this what Rust is for?

The message here I'd like to convey is not to stop the ergonomics initiative ‒ I
don't think being unergonomic without a good reason is a nice thing, but to
accept that ergonomics has downsides and needs to be considered with the
advantages. Rust is the language that's being put into rough places. I'd rather
it gives me all the heavy and uncomfortable weapons I need, than to try to calm
me down that there are no monsters in the cellar. I'll take a never-ending
stream of paper cuts over one atomic bomb in a blue moon.

If I ever commented on *your* proposal with something like „But what if this
really arcane set of conditions meet, and a new version of dependency does that
at the same time, won't this cause *BOOOM*?“, then it's just me being scared of
atomic bombs. Because I have a very good talent of getting myself into these
arcane conditions. It's an attempt to help fix the proposal so the bugs don't
crawl through any cracks.
