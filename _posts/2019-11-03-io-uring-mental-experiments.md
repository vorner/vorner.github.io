---
title: "Mental experiments with `io_uring`"
---

# Mental experiments with `io_uring`

Recently, a new Linux kernel interface, called `io_uring`, appeared. I have been
looking into it a little bit and I can't help but wondering about it.
Unfortunately, I've had only enough time to keep thinking and reading about it.
Nevertheless, I've decided to share what I've been thinking about so far in case
someone wants to write some actual code and experiment. Basically, I have an
idea for a crate and I'd love someone else to write it 😇.

Therefore, as a little disclaimer, all things included here can be inaccurate or
wrong. I believe the general high-level picture is correct, but as I haven't
actually used the APIs, all kinds of surprises might appear. The code examples
are not tried either and are just illustrative.

## A little about the kernel API

There are some APIs for asynchronous network IO in Linux. The older and more
portable ones are `poll` and `select`, the newer one being `epoll`. These allow
stuffing the kernel with large amount of sockets and waiting until some of them
are ready to read or write data. Once the kernel identifies a subset of them as
being ready, the application performs the reading and writing operations.

But these don't really work for files. There's the AIO subsystem, but it is
generally considered awkward to use and with huge amount of drawbacks and nobody
really uses it at all.

However, there's a lot of applications that could benefit from having
asynchronous and cheap file access. The cheap part is also important. On one
side, syscalls (eg. calls from userspace into the kernel) got more expensive
with spectre workarounds. On the other hand SSDs and NVMes are getting much
faster and are capable to work more in parallel (a rotation disk could do only
one read at a time, SSD can read simultaneously in each of its chips). While it
didn't matter that much for rotation disks (because waiting for the disk itself
took so long nobody cared about the syscall cost) and network IO is generally
quite expensive too, the syscalls form significant part of modern disk access.

The `io_uring` tries to solve these problems. While it can be used for
asynchronous network IO, the main purpose is the disk IO.

The interface is composed of two ringbuffers or queues, one for requests going
from program to kernel, the other for the kernel returning results. The
queues live in memory shared between the kernel and the application, so reading
and writing these queues don't need any syscalls, only some thread
synchronization primitives. There's one syscall that is used both for notifying
the kernel that there are at least `n` new requests in the queue and that the
application would like to wait for at least `m` responses to be ready before
continuing. This allows one to submit many requests and receive many responses
per one syscall, amortizing its cost. Under some circumstances one can avoid
even that one syscall.

Comparing to other kernel interfaces, it is quite well documented. Which still
means somewhat worse documentation that most of Rust crates, but there's more
documentation that just the kernel sources (I'm looking at you, netlink…).
There's even a [high-level document](http://kernel.dk/io_uring.pdf) about it.

## What exists already

Rust and rustaceans being what they are, if you say „performance“ you can expect
few crates to pop up. Indeed, I've found some and [this
one](https://docs.rs/io-uring/0.2.0/io_uring/struct.UringQueue.html) looks the
most user friendly.

However this is still a bit low-level. If we let aside that the kernel API now
allows much more than just reading and writing, using this manually in normal
code will turn out tiresome. It's a useful wrapper of OS API in a similar way as
[`mio`] is a wrapper around `epoll` (or something else on other platforms).

But similarly to how [`tokio`] builds asynchronous network primitives on top of
[`mio`], we would like to be able to use asynchronous files through this new
API. Let's look where the challenge lies and let's also look if we can abuse the
`io_uring` for something besides files.

## Design goals

First, we want good performance. The design of the kernel API goes to great
lengths to eliminate costs as much as possible. If we are going to play with it
in Rust, it would be nice to know how far we can stretch it on the userspace
side too. So, we'll try to avoid unnecessary copies, allocations, context
switches…

Second, the interface should be usable without any `unsafe`. We may need to use
some of it internally, but people writing databases or fulltext search engines
or whatever else that needs really fast disk IO should not have to worry about
that kind of stuff. What they do is already challenging enough without piling
more onto them.

And third, the API should be comfortable to use if at all possible.

Things like being sound (not being able to cause an undefined behaviour without
using `unsafe`) go without saying.

## Ownership of buffers

We are coming to the first challenge out there. We want to be able to do
something like this:

```rust
async fn get_cache(path: &Path) -> Result<Vec<u8>, Error> {
    // This line can probably block, but let's ignore it for now
    let file = File::open(path)?;

    let mut buff = vec![0; 1024];
    let amount = ioring_read(&file, &mut buff).await?;
    buff.truncate(amount);
    Ok(buff)
}
```

This, however, is problematic. When requesting the read, we pass the buffer to
the kernel. It'll do the reading directly into the buffer and will tell us when
done. But there's no way to take the buffer away from the kernel sooner than
that. For one, the `io_uring` doesn't seem to have a way to cancel an ongoing
operation ‒ disk access is fast enough not to need timeouts and such. But even
if there was, it would probably work by sending another request „please cancel
that other request“ through the queue and having to wait for confirmation to
get out of the other queue.

*Edit: As pointed out by glaebhoerl in the comments, `io_uring` now has a
[support for cancellation](https://git.kernel.dk/cgit/liburing/commit/?id=8b68609d67f481c5df7bee990234bf87be382514).
But the cancellation works as I expected, by sending the requests through the
queues. Therefore this problem is not solved by it.*

Why is this a problem? Because futures in Rust can be cancelled by dropping at
any time. And even if we placed the waiting into the destructor, we can't really
count on the destructor being run. After all, it's always possible to forget the
future:

```rust
// What's the exact syntax for a select or timeout right now?
match ioring_read(&file, &mut buff).timeout(Duration::from_secs(0)).await {
    Ok((amount, _timeout)) => buff.truncate(amount),
    Err(read_future) => mem::forget(read_future),
}

// The `&mut` died with the future without descructor, so we can now:
println!("{:?}", buff);
```

Now, if the read times out, we don't run the destructor. The kernel still thinks
we want it to put data into `buff` and at some point it's going to do so. But we
print the content of it at the (potentially) same time. I don't think anyone
explicitly said what happens in such case, because the kernel is not our (Rust)
thread and the Rust threading/aliasing/data-race model may not be directly
applicable. But let's assume its some kind of undefined behaviour that we
definitely don't want.

Destructors won't help. Pinning won't help (pinning doesn't stop us from reading
from it).

So far I think there are three options:

* Use a separate buffer to submit to the kernel. Once the kernel tells us
  everything's done, we copy the data to the buffer provided as parameter. If we
  do that copy inside the future's `poll` method, we are sure it's still alive.
  But that's an extra copy. Considering how much effort was put to minimise
  these kinds of overheads, it would be a pity to break the zero-copy
  characteristic of the API.
* Pass the ownership of `buff` into the future and get it back as a result. If
  we `forget` the future, the buffer goes down the black hole too so we can't do
  anything evil with it. But that is kind of uncomfortable to use.
* Have our own buffer type. It would „lock“ itself on submitting it into the
  kernel and „unlock“ once it gets back out. If we ever tried to access it while
  locked, it would panic. Having to use some other type than a `Vec` (or slices)
  is somewhat uncomfortable, but probably better than the above. On the other
  hand, it incurs some (probably small) performance cost for checking it is not
  locked. Maybe if it was just a wrapper around whatever other type and the
  inner buffer could get extracted (when unlocked) it would be cheap enough.

I'm slightly inclined to the last option. It has another benefit. The `io_uring`
has a mode in which it prepares some buffers beforehand and they work a little
bit faster. Using a special type would mean we could allocate them as part of
these prepared buffers.

## Reading the completion queue

We've created our future that represents the request to the kernel to read some
data. We may put the request to the kernel at many different occasions ‒ at
least on creation of the future (that's not how Rust futures usually work, but
it could be done) or on the first poll of the future.

But eventually someone needs to call „I've submitted n tasks and want at least m
tasks done from you“ syscall, which might block, and then handle the done tasks
somehow.

There are two obvious solutions.

* Start a separate thread for that. The problem is, our future probably runs on
  some executor. As it likely does some network IO as well, let's assume its the
  tokio executor. So once our disk IO is done, we need to wake the future and
  it'll be woken in another thread. Having a separate reactor thread turned out
  to be too slow for tokio itself and it was *network* IO where the relative
  cost of the thread wakeup was smaller. Or, considering it from a different
  point of view, we've eliminated a lot of syscalls (we are able to get bunch of
  reads per one syscall) but added a context switch, which is probably at least
  as expensive (because to do a context switch, the processor needs to transfer
  to kernel and back to userspace).
* Ask tokio to integrate the support for our `io_uring` directly into itself.
  While technically possible (the `io_uring` AFAIK can act as an epoll
  replacement too), I don't think the devs would be overexcited at dumping an
  experiment like this with platform-specific API into their production code. So
  while not ruling this option out completely, it's probably not happening any
  time soon.

Is there another option? I think there is. Tokio is quite flexible (that's why I
use tokio as example here, not eg. `async-std`, because the latter won't let us
do the kind of abuse we want here). Each thread participating in the runtime has
three ingredients (no matter if it's the `current_thread` or the work-stealing
runtime or even some completely different one):

* The executor. Its job is to keep track of tasks (the name of futures that
  were spawned and started to get executed) and running the ones that are ready
  to make some progress.
* A timer. Its job is to wake up futures that asked for being woken up at
  certain time.
* A reactor (or, I think the new version calls it driver) that waits for the
  kernel to notify it about sockets (or any other file descriptors) being ready
  to be read or written.

Whenever a layer doesn't have anything to do right now, it passes control to the
layer below it. So, if the executor doesn't have any ready futures, it passes
the control to the timer. It checks the clock and its data structure and maybe
wakes some futures, in which case it would return the control back to the
executor to run them. If there were none to be woken, it would compute for how
long there'll be none such and pass the control to the reactor. The reactor
would extract notifications from the kernel (or block on getting them) and wake
the relevant futures.

The trick here is, this passing of control is abstracted using a [`Park`] trait.
And on top of that, we can assemble the runtime ourselves, doing any necessary
tweaks to it. There's a [documentation][runtime-tutorial] how to do it (for the
0.1 tokio, but I assume it'll assume it's still possible with the 0.2 branch,
probably with some minor differences). This allows us to insert additional layer
into the equation.

We can put the `io_ring` poller between the timer and reactor. The reactor
implements [`AsRawFd`] and, unless my guess is very wrong, it would become
readable if the `epoll` inside has some events. The `io_uring` can be used to
watch readiness of a file descriptor, so we can plug it in. When the completion
queue spits out finished reads or writes, the corresponding future can be woken
up. If the `epoll` file descriptor is returned, we can do one turn of the
reactor.

We could probably connect it the other way around ‒ the completion queue can
have a companion file descriptor that's readable if there are finished tasks to
pick up. Then we could simply create some kind of driving future that processes
all the completed tasks when woken up. But my gut feeling is that the disk IO is
going to be more sensitive to latency and performance and therefore should be
the one closer to the top.

## Using it for network IO

The `io_uring` was designed mainly for disk IO because that was the missing
functionality. It was already possible to effectively handle large amounts of
network IO operations so the gains for using the new interface for networked IO
isn't much.

But there probably can be some. But first, we probably don't want to register
reads and writes directly as with the files. Unlike files, network sockets might
take really long time to provide or send data or they may not do it at all. So
such task would be sitting in the kernel a very long time and a lot of them
would be submitted. I didn't find what the limits for how many tasks can be
submitted is, but there was a hint about problems when the completion queue
fills up ‒ which can happen if there are many such tasks.

We can, however, use the queues to replace `epoll`. That could save some
syscalls ‒ for example, we could register multiple file descriptors using just
one syscall. It would also piggy-back on some disk IO.

Furthermore, when `epoll` turns once, it may spit out multiple sockets ready to
perform some operations. Currently, each future performs that operation on its
own, each performing another syscall. All these follow-up syscalls could be
batched and submitted at once through the queue.

So, all in all, this could help reduce number of syscalls and may improve
performance a little bit. This would need to land in tokio itself, though (I
think I've seen a pull request for something of it somewhere, but now I can't
find it).

## What's next?

I don't know if any of these ideas are any good and I don't have the time to
pursue them right now. But they sound like something worth at least trying out.
So, if anyone wants to have a go at it, here I give all the ideas for free. I'll
happily help brainstorming further ideas and provide advice, I just don't feel
like investing time into the actual coding.

[`mio`]: https://crates.io/crates/mio
[`tokio`]: https://crates.io/crates/tokio
[`Park`]: https://docs.rs/tokio-executor/0.2.0-alpha.5/tokio_executor/park/trait.Park.html
[runtime-tutorial]: https://tokio.rs/docs/going-deeper/building-runtime/
[`AsRawFd`]: https://doc.rust-lang.org/std/os/unix/io/trait.AsRawFd.html
