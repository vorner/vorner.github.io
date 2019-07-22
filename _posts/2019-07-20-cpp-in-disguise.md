# Is Rust C++ in disguise?

I'm going to give a talk about Rust to some C++ folks. And while preparing it, I
have to wonder about one thing.

Sure, Rust and C++ have different syntax ‒ C++ has function return types on the
left side (most of the times), Rust has them on the right side. Rust has the
borrow checker. C++ has template madness. They differ in their error messages.
But if you strip the language of the clothes of syntax, and leave the naked
semantics inside, how much do you have to squint to claim they are the same?

And I believe you don't have to squint that much. This is IMO a good thing while
introducing a C++ developer to Rust. The C++ developers are wary of new
languages that claim to be system ones, because a lot of them aren't, or not
to the same level as C++ is. Having the same semantics under the hood kind of
says that yes, Rust can do all or most of the stuff C++ can, with the same or
similar performance characteristics.

## Few disclaimers

I don't know if any reader will find this post of any value. Well, I never know
upfront, but here this is mostly me trying to sort my own ideas a bit and while
doing so, I might as well do it here publicly.

If I make some claims that are blatantly not true, I'm all happy to be
corrected. However, this is going to be mostly on the *intuitive* level, so
there are going to be some simplifications. Please don't get upset about little
inconsequential details like C++ and Rust having different calling conventions
or different name mangling scheme. Some squinting is still allowed. I'm also not
providing references to exact sections in rustc codebase or C++ standard to back
up the claims.

Second, I'm going to talk about *mainstream* C++ implementations. There's one
canonical implementation of Rust (I know about `mrustc`, but I'm not going to
talk about that one either), but C++ is a standard. That standard allows *a
lot* of leeway for implementations so one could create a C++ *interpreter* or
compile to JVM or mostly whatever (and as crazy as people are, someone probably
already *did* create a C++ compiler targeting JVM). The point is, when we talk
about C++, we talk about the usual ones that compile to native code, like GCC,
Clang or MSVC.

By the way, *most* of what this talks about is also applicable to C ‒ not
everything, for example C doesn't have stack unwinding, but most things still
apply.

## The execution model

Both languages are ahead of the time compiled to native. This erases most of the
type information (which is preserved in optional debug info, but not actually
used by the program itself), the only thing left after types is the actual
actions taken with the bytes in memory. There are small exceptions ‒ RTTI,
tables of virtual methods, etc. But they are not used globally, only when
absolutely necessary. Non-virtual methods are a syntax sugar for quite an
ordinary function with the first argument being the object the method is called
on (`this` one is implicit in C++, but it is there). Virtual ones are just a bit
of glue code that looks up in the virtual method table and calls that (basically
a call by a pointer to function).

Speaking about types and memory, types compose in a flat manner ‒ a field in a
struct is part of the memory image of the whole struct. Any indirection in the
composition is explicitly asked for by the programmer. The fields are identified
only by offset from the start of the struct, which is hardcoded into the
generated code, not by eg. names.

Memory is split into three kinds (+some minor niche cases like thread local
storage, but let's skip that one):
* Static storage: these are global variables and similar. Basically, the things
  that live for the whole lifetime of the program.
* Explicitly managed heap: the programmer has control over when the memory is
  allocated and freed. It is often with a lot of help from libraries and the
  RAII pattern, but all this still translates into series of calls to the
  allocator (`malloc`/`free` analogs). It is also possible to call these
  manually, but is generally not encouraged. Memory thus allocated doesn't move
  *on its own* (garbage collectors in other languages are allowed to move
  objects from place to place to combat memory fragmentation, because they can
  fix the pointers to them; all object moves in both C++ and Rust are done only
  as direct results of programmer code ‒ if an object moves, there's a line of
  code this can be attributed to).
* A separate stack for each thread of execution. This stack has a limit on its
  size and is non-relocable (can't grow beyond the preset limit and can't move
  from place to place). It is continuous (not segmented, though I'm not sure if
  the C++ standard would allow for segmented stacks).

Threads map directly to the OS-level threads (though libraries can provide
further abstractions above that).

## Runtime

Some people say that neither Rust nor C++ has a runtime. This is slightly
misleading.

It doesn't have a fat runtime like Java or Python or Go does that oversees the
run of the program. It doesn't have a garbage collector that puts the whole
program to sleep whenever it likes. It doesn't have a JIT that modifies the
instructions as the program runs.

But both Rust and C++ have a runtime. There's something that prepares the
program to run before `main` is started. There's something that takes care of
loading dynamic libraries. There's the memory allocator to manage the heap.
There are things like stack unwinding when an exception/panic happens.

The crucial difference is, all of this is on-demand. If you don't allocate
memory, the memory-allocator doesn't get used. But it's not true that if you
don't allocate in Java, you'd be guaranteed the GC would not run. The runtime
doesn't oversee the program execution, it obeys the program.

This on-demandness of the runtime makes it possible to *easily* embed both Rust
and C++ into other languages, even ones with the overseeing runtime. However,
having *two* overseeing runtimes in the same program is probably asking for big
trouble (I'm reluctant to claim it's impossible because someone would just put
in the effort to prove me wrong).

Furthermore, in both C++ and Rust, it is possible to opt-out of various parts of
the runtime. You can give up the memory allocator if you aim to be the kernel or
to run on an embedded device. Of course, you also lose big parts of standard
library if you do so.

## Undefined behaviour

And yes, both languages have the notion of undefined behaviour. While it is not
possible to invoke it from Safe Rust (barring compiler bugs), this barrier is on
the syntax level.

Edit: As kindly pointed out by `etareduce`, the things that are considered UB
are not completely the same ‒ signed integer overflow is UB in C++, while `bool`
with value 2 is UB in Rust (though there's significant overlap, like accessing
dangling pointers).

## Does it mean C++ and Rust are the same?

Well, no, of course not :-). The syntax is obviously a big part of the story.
Having a static analyser (the borrow checker) a mandatory part of compilation is
also something important. Rust has some modern tooling. And it drops *a lot* of
old historical baggage (I'm looking at you, array to pointer decay from C!).

There are a lot of resources on the Internet about how these languages are
*different*, so I'm not going to go into that topic right now.

But I still like to say that Rust might be what you get if you decide to do C++
today from scratch. Not that it knew from its start it wants to become something
like C++, that AFAIK kind of happened on the way.
