# Saving some allocations

I guess most of you know the feeling that some code is not completely optimal.
It probably doesn't matter much in practice, because there are more expensive
things around somewhere else, but the feeling when looking at it and thinking
„well, there must be some better way“. And then you go on a war with the code to
come up with something that is hopefully still readable, but somewhat better in
whatever metric you're obsessing with at the time.

I often declare these kinds of wars on unneeded copies and allocations.
Talking to the global allocator is not exactly cheap and making extra copies is
just not elegant. Sometimes it means using a [bump allocator][bumpalo], or even
writing an [extension for it][bumpalo-herd]. Sometimes it means extensive use of
borrowing. Now I have a long-standing battle with few of these allocations in
a service I work on in my day job. There seems to be a possibility this one will
finally get resolved and here goes a description how.

## What the code does

The service accepts requests over HTTP. The POST body is protobuf encoded.

The HTTP part is taken care of by [hyper]. For the protobuf, I've chosen
[prost], mostly because it makes the generated API look more like Rust than Java
and because it Just Works without any hassle around installing plugins. And it
seems popular too, which means it hopefully won't fall into disrepair any time
soon.

The code currently looks something like this (I'm replacing the specifics with a
phone-book example and simplifying):

```protobuf
message Entry {
    optional string name = 1;
    optional string phone = 2;
    optional string address = 3;
}

message Batch {
    repeated Entry entries = 1;
}
```

```rust
async fn phonebook_submit(req: Request<Body>, limits: &Limits)
    -> Result<Response<Body>, Error>
{
    // Accumulate the whole body. But protect from evil or broken clients that
    // send too much stuff, we don't want to eat all the RAM.
    let mut data = Vec::new();
    let mut body = req.into_body();
    while let Some(chunk) = body.next().await {
        let chunk = chunk?;
        if data.len() + chunk.len() > limits.max_bytes {
            return Err(TooLarge.into());
        }
        data.extend(chunk);
    }

    let decoded = Batch::decode(&data)?;
    if decoded.etries.len() > limits.max_entries {
        return Err(TooLarge.into());
    }
    process_submit(&decoded)?;
    Ok(Response::builder().status(StatusCode::NO_CONTENT))
}
```

This code works. But I have this itching that it allocates and copies data
around needlessly. Let's look where.

## Collecting the whole HTTP body

First it accumulates the whole body into a `Vec`, because [prost] doesn't allow
parsing incrementally (it can [`merge`], but this won't handle a field spanning
across the buffer boundary). So every time we receive a request, new `Vec` is
allocated and the data are copied into it. This allocates at least one, but may
reallocate to accommodate longer multi-chunk requests.

In practice, most requests are short and fit into a single chunk. And [prost]
doesn't need a `Vec`, using the chunk directly would be fine. The type of the
chunk is [`Bytes`], which implements [`Buf`], which is what the parser wants. So
we could somehow check if the body is indeed in a single chunk and have two
branches of the code, one that uses it directly and the other handling the
longer bodies with a `Vec`. But that would make the code harder to read.

We could at least reuse the `Vec` between requests and not allocate every time.
But we would still be copying the bytes around.

The [`Buf`] trait allows implementations to be non-continuous. That is, it
doesn't have to be one big slice, it can be a sequence of shorter ones. So if we
could instead just chain the chunks and create a one [`Buf`] that represents
their sequence, everything would be nice and shiny.

The [aggregate] function in [hyper] actually does pretty much that, but it has
three disadvantages:

* It returns `impl Buf` instead of concrete type. [hyper] in general has the
  habit of not wanting to give me concrete types, which makes it harder to pass
  the values around or store them (it pollutes code with generics, requires
  dynamic dispatch and boxing in traits ‒ which is *another* allocation I don't
  want to have).
* I want to have that defensive check of how much data I've received from the
  client and abort it before all RAM is consumed. It seems the [aggregate] will
  just consume everything, no matter how large the payload is. Checking some
  kind of `Content-Length` header first doesn't help, because with a chunked
  encoding it is not mandatory. If one uses [aggregate] in the server, a
  malicious client can just kill the server by sending too much data.
* It doesn't provide the other optimization I describe below at the moment (I've
  went to check the code). Though I might submit a PR to fix this, it should be
  quite simple.

For the above reasons, I've decided to roll my own type that is basically a
bunch of [`Buf`]s chained together (well, actually, I have two slightly
different ones) and publish it in the [bytes-utils] crate. See the
[`SegmentedSlice`], which can be built on top of some [`SmallVec`] of chunks.
That way if only few chunks arrive (we expect just one), we don't do any extra
allocation on top the one that [hyper] does to create the chunks.

```rust
let mut chunks = SmallVec::<[Bytes; 4]>::new();
let mut body = req.into_body();
while let Some(chunk) = body.next().await {
    let chunk = chunk?;
    if data.len() + chunk.len() > limits.max_bytes {
        return Err(TooLarge.into());
    }
    chunks.push(chunk);
}
// This one doesn't allocate and will present a Buf view into all the chunks
// we accumulated.
let data = SegmentedSlice::new(&mut chunks);
// And we can pass that do prost directly.
let decoded = Batch::decode(data)?;
```

## Allocations to fill the protobuf messages

By default, the structures generated by [prost] will look something like this
(I'm not including bunch of attributes it puts on them, they are not interesting
right now):

```rust
struct Entry {
    name: Option<String>,
    phone: Option<String>,
    address: Option<String>,
}

struct Batch {
    entries: Vec<Entry>,
}
```

Each non-empty `Vec` or `String` is another separate allocation. In the case of
the strings that come from the input, they are just copied from there. In theory
[prost] could also borrow from the input data ‒ do something like this:

```rust
struct Entry<'a> {
    name: Option<&'a str>,
    ...
}
```

Then it could just point into the input data instead of allocating and copying.
But that would make working with the structures and the whole API more hairy, so
the suggestion to allow that was turned down.

Nevertheless, a different feature is in progress. At the bottom, there's the
[`copy_to_bytes`] method from the [bytes] crate. If the type this is called on
is [`Bytes`], it'll not allocate new instance and copy the bytes. The [`Bytes`]
type is something like `Arc<[u8]>` on steroids and it can hand out owned
sub-slices of itself (increment the reference count, but point to subset of the
original allocation). So `Bytes::copy_to_bytes` will just subslice itself and
hand out that without copying. That's pretty cool.

And [prost] can take advantage of it, with the [`bytes`][prost-bytes]
configuration method. For now this works only for `bytes` fields in the
protobuf, but it seems some form of support is coming for strings too.

OK, so if we fast-forward to some future release of [prost], we will likely get
this struct instead (if we explicitly ask for it):

```
struct Entry {
    name: Option<Bytes>,
    ...
}
```

Then, when parsing, it'll call this [`copy_to_bytes`] method. If the original
input is something like `&[u8]`, it'll just copy the bytes into a new
allocation, but if the original input is also [`Bytes`], it'll just slice &
increment the reference count instead, which is cheap. Almost like the original
idea with borrowing from the input, but without any annoying lifetimes around.

But, isn't our input some kind of [`SegmentedSlice`]? It is, but I've taken the
care to also support the same optimization, at least where possible. If the
whole request (one call to [`copy_to_bytes`]) can be fullfilled by a single
buffer, then it delegates to that one. If the underlying buffers are [`Bytes`]
(like in our example above), it'll use their optimized version. Unfortunately,
if the request is across a buffer boundary, then there's no option but to
allocate & copy. However, given that we usually expect to have just one chunk
(so there aren't multiple buffers to start with) and even if there are, each
boundary can "kill" only one string/byte string, we have improved the situation
by a lot.

## But `String` is more comfortable to use than `Bytes`

If the generator can create structures with [`Bytes`] in it, it helps the
allocations. But that loses all the fancy string manipulation methods, like
[`lines`], because it deals with arbitrary byte arrays.

For that reason, the crate I've publish also contains the [`Str`] type. It's a
thin wrapper around the [`Bytes`] type, but dereferences to `&str` and puts all
the methods back.

Furthermore, it adds few methods of its own. Many of the string methods return
sub-slices of `&str`. But these are borrowed and therefore introduce lifetimes
into whatever structures they are put into. So there are few analogue methods
(like [`lines_bytes`]) that return more instances of [`Str`] instead of `&str`.
These are still cheap to make (slightly less cheap than the `&str`, but it is
only one reference count away) while providing an owned value.

I don't know if [prost] will eventually allow some way to fully customize which
types it should generate ‒ that would, additionally, allow using some `SmallVec`
instead of `Vec` for `repeated` fields, to save allocations in case only few
values are present. If it would, it should be possible to have structs with
`name: Option<Str>` in them. If not, it is still possible to create the `Str`
from `Bytes`, even though that would be less convenient.

## Does it help?

Since the part where [prost] replaces `String` with `Bytes` isn't available yet,
I haven't integrated this into the service. So I don't know if it makes it in
any way measurably faster. But considering the processing part, after it is
received and parsed, is probably much larger than this, I don't think it'll be a
big improvement. But it'll certainly make my itching about number of allocations
get better 😇. After all, who does performance optimizations because of actual
speed?

That being said, this isn't necessarily an obvious win. The [bytes] crate is a
treasure box of various little clever tricks (if you use the crate, I really
recommend reading through its documentation). One of them is, if you use the
[`BytesMut`] type to first generate the data (for example by reading them from a
socket) and then chop off the generated buffer in the form of [`Bytes`], you can
keep reading into the same [`BytesMut`]. Under some circumstances ‒ like, if all
instances of the [`Bytes`] that was handed out were dropped and you call
[`reserve`], the original, chopped-off part of buffer is reclaimed and can be
reused. So in the old code, if we keep the `Vec` around, or if it grows
exponentially and has enough capacity most of the time, the bytes are copied
over, but a new allocation is not done, the same buffer is used. In the new way,
we keep storing the instances of [`Bytes`] around, so they can't be reused. So
every now and then the [`BytesMut`] buffer runs out and a new one needs to be
allocated. However, if we expect only one chunk to arrive, this doesn't happen
(there's no chance to reuse anything).

Furthermore, if there was a big buffer to start with, and we have sliced a small
[`Bytes`] slice out of it and keep it around, that keeps the whole original big
buffer allocated. This could bite us if we actually store for example one small
string out of the whole request in some kind of semi-permanent storage instead
of throwing everything out after processing. That way our memory consumption
would grow by all that unused but not returned buffer space. It's not really a
memory leak in the strict sense ‒ there still is a pointer to the original
buffer and if we drop the small string, it is properly deallocated. But it would
waste the space nevertheless.

So I think using [`Bytes`] and the utilities in the [`bytes-utils`] crate can be
a double-edged sword. There are situations where it (at least theoretically)
could help when used right. But there are also situations where it could hurt.

[hyper]: https://hyper.rs
[prost]: https://crates.io/crates/prost
[`Bytes`]: https://docs.rs/bytes/1.*/bytes/struct.Bytes.html
[`Buf`]: https://docs.rs/bytes/1.*/bytes/trait.Buf.html
[aggregate]: https://docs.rs/hyper/0.14.*/hyper/body/fn.aggregate.html
[bytes-utils]: https://docs.rs/bytes-utils/
[`SegmentedSlice`]: https://docs.rs/bytes-utils/0.1.*/bytes_utils/struct.SegmentedSlice.html
[`SmallVec`]: https://docs.rs/smallvec/1.*/smallvec/struct.SmallVec.html
[`copy_to_bytes`]: https://docs.rs/bytes/1.*/bytes/trait.Buf.html#method.copy_to_bytes
[prost-bytes]: https://docs.rs/prost-build/0.7.*/prost_build/struct.Config.html#method.bytes
[bytes]: https://docs.rs/bytes
[`BytesMut`]: https://docs.rs/bytes/1.*/bytes/struct.BytesMut.html
[`reserve`]: https://docs.rs/bytes/1.*/bytes/struct.BytesMut.html#method.reserve
[bumpalo]: https://docs.rs/bumpalo
[bumpalo-herd]: https://docs.rs/bumpalo-herd
[`merge`]: https://docs.rs/prost/0.7.*/prost/trait.Message.html#method.merge
[`lines`]: https://doc.rust-lang.org/std/primitive.str.html#method.lines
[`lines_bytes`]: https://docs.rs/bytes-utils/0.1.*/bytes_utils/string/struct.StrInner.html#method.lines_bytes
