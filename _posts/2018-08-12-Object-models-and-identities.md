# Object models and identities

In many things, Rust is very much like C++. It's memory management strategy is
mostly the same, threading models are copied vanilla, both compile to native
code and do about the same optimisations at that time, and traits and templates
have a lot in common too. Both tend to be rather feature-rich languages with
quite a lot to learn. While Rust is definitely better teacher (I'm looking at
you, C++ error message!) and has many more „safety covers“ over the dangerous
moving parts inside the engine, the design of the engine is more of an evolution
from C++ than a completely new thing.

But I've noticed one rather subtle difference in the philosophy of the
languages I'd like to describe here. To make it somewhat more complete, I'll
also throw what some other languages do in this area in.

## What is an object and what is an identity

I'm not going to bring formalism here. For one, I don't think it fully exists to
describe this thing. For another, while formalism is important for the computer,
we humans prefer to hold languages by the more intuitive end and this is more
about how one reasons about the language.

By object I mean *something* stored in memory ‒ be it one integer, a struct
instance, an enum instance, an instance of class in case of C++ (actually called
an object) a dict dressed in fancy OOP coat in Python or Lua (called object). It
can be composed from further smaller objects. But it is something the brain can
take hold of and manipulate while thinking about it.

An identity is an abstract notion that allows us to tell one object from
another. For a computer, an object is bunch of bytes somewhere in the memory and
it doesn't really care further. But our mental processes do care ‒ and we can
track one living object across time even though the bytes representing it change
(in their value or possibly even in their number and location in memory). We
usually track the object from its creation to its destruction (and in a well
behaved program, every object is created and destroyed exactly once, but
forgetting to destroy some of the objects is sometimes considered acceptable).

Of course, there are situations where it becomes somewhat fuzzy. What if I take
an object and send it over a network to another process in some kind of
distributed computing framework? But let's just pretend for a while the world is
a nice place to live in and such oddities don't happen.

There are also some primitive things we don't really assign identity to. For
example, an integer doesn't have much of a identity ‒ we don't care if two
integers with the value `4` are considered one and the same four or two distinct
fours that just happen to have the same value. We make the distinction for some
more complex things ‒ if we have two `DatabaseConnection` objects, we consider
them distinct even though they were created by the same means and compare equal
(if we even bothered to define equality for them). Languages (and people) draw
the line at different places and it is kind of fuzzy (Does a string have an
identity?).

## What identifies an object in C++?

An object in C++ is identified by it's address in memory. An object is born on
some address and it dies on the same address. There can be no unrelated object
at the same address during the lifetime of this one (parts of the object that
are also objects on their own can live on the same address,  but it's like
saying that your brain shares the same place with your body). This is quite a
simple answer to a complex question and as such has some consequences for the
language.

A copy constructor creates a new instance of the object that happens to be
somehow similar to the original one. And copy constructors happen invisibly at
many places ‒ like when passing something as a result out of a function.

In an assignment, the left-hand object still stays the original left-hand
object, but it modifies itself to resemble the right-hand object. And this is
the default behaviour of the language ‒ to create new instances of objects all
over the place and destroy them shortly afterwards. There's no way an object can
change its place.

What about move constructors and assignments? Well, these are a bit of a
cheating. A move constructor is in principle only a copy constructor but it is
allowed to scavenge parts of the original to build itself. However, a new object
is still created and the original is not destroyed by this action. The original
is left hollow in place, but it is still a valid object, even if not very
useful. The destructor for the original will still be run whenever that variable
falls out of scope.

If we look at this code, the `a` vector at the end is still a vector, it'll just
happen to be empty.

```c++
vector<int> a;
a.push_back(1);
a.push_back(2);

vector<int> b(move(a));
// a is empty here, but it still is a vector
```

Or, if explained by a car analogy (of course it was coming), a copy constructor
is looking at one car and building a brand new car that looks just the same. A
move constructed car is a brand new car, but using the seats and engine of
the old one, while leaving the rest of the old one in place ‒ and letting
someone else to take care of either disposing it or furnishing it with new
seats. But these cars just don't know how to drive from one place to another.

## Rust and object identity

In Rust an object may live on different addresses during its lifetime (I don't
mean the borrow checker's lifetimes, but the time between creation and
destruction) ‒ though only at one address at a time. Furthermore, it can happen
that two unrelated objects live on the same address (for example objects that
take 0 bytes in memory ‒ which is something that is not really allowed in C++,
precisely because C++ needs the address for object identity).

This code can print two different addresses or the same addresses as the
compiler sees fit. It actually does ‒ try running it in the playground in both
debug and release mode. In release mode, it produces the same values though by
any notion of identity `x` and `y` are two different objects (they even have a
different *type*).

```rust
struct X;
struct Y;
let x = X;
let y = Y;
println!("{:p} {:p}", &x as *const _, &y as *const _);
```

While Rust has something akin to C++ copy constructors (the [Clone] trait), the
default thing that happens to objects is to move them around. It's an activity
during which the object can change its address (it doesn't have to). The object
itself is not notified of this move ‒ it can't take any action and the compiler
is allowed to implement the move just as a `memcpy` of the object's memory image
between two places ‒ while the old place holds the same image, it no longer
represents a valid object.

Once again, this code prints different values in debug mode and the same ones in
release:

```rust
struct X;
let x = X;
println!("{:p}", &x as *const _);
let y = x;
println!("{:p}", &y as *const _);
```

Even after moving the object, possibly to a new address, it is still the same
object. It's like taking the car and parking it somewhere else ‒ no new one
created, and no destructor runs on the old car ‒ the old variable name just
ceases to refer to anything meaningful.

References can't be kept across the move. But that doesn't necessarily go
against the idea of the object being the same one ‒ if a reference is something
like identification of a specific parking spot, you no longer can refer to the
same car as „The car at place 72“ when it moved ‒ it's somewhere else by now.

Assignment is just a destruction of the previous object in the left-side
variable and moving the right-side object into its place (well, it takes a bit
of thinking to prove the same place in memory generally needs to be reused).

From another point of view, Rust gives names to values (and calls them
„bindings“), while C and C++ gives names to pieces of memory (and calls them
„variables“). Pointers in C++ point to bits of memory and references in Rust
point to the objects there.

From an intuitive point of view, this behaviour seems much more natural to me
(real objects, like cars, usually move around, don't get created in a new place
anew and the old ones demolished). I believe these little details are more
potent in preventing bugs than the safe/unsafe distinction, for example. After
all, many programmers of modern C++ don't touch raw pointers either ‒ they know
these are dangerous even without the `unsafe` warning signs. But having the
language act like something our subconsciousness knows how to handle gives us a
huge benefit ‒ most of the thinking happens on the subconscious level.

## What about other languages?

First, there are some very low-level languages, like Assembler or C. These have
only tools to name bits of memory. They don't introduce any paradigms, like
constructors and destructors, that would require the language to take a stance
in regards object identity or lifetime, or to even say what an object is. The
programmer there is allowed to introduce any notion as seen fit. It's like not
really being able to talk about whole cars, only about differently sized and
shaped pieces of material ‒ it's up to you to call it a car if you so like.

And then there's this really big family of higher-level languages usually with
garbage collectors. The object identity is tied to the value, not the address ‒
some of the languages don't even provide a way to find out an address and some
of them feel free to move the objects around in memory as their garbage
collector sees fit. So in that way it is somewhat similar to what Rust does.

However, the difference there is, the programmer doesn't really hold the object
itself like in Rust. The only thing they have access is some opaque reference to
the object. While there is something like the copy constructor (often called a
deep copy), it doesn't allow to directly have the object, only talk about it.
These references don't have identities of their own and can therefore be just
copied around ‒ and no matter how many references talk about the same object,
the object lives somewhere undisturbed by such thing.

Of course, there are languages that don't really fit naturally into any of these
groups. Some languages don't really have any notion (or have a very weak one) of
order of execution or time, so it makes little sense to talk about lifetime of
something (Prolog). Perl is a bit like C in a way it doesn't really take a
stance about anything and lets you do whatever you like, but it doesn't give you
direct access to bits of memory either (on some level, it gives you access to
bits of text and everything in Perl is text to some extent ‒ at least
conceptually).

[Clone]: https://doc.rust-lang.org/std/clone/trait.Clone.html
