---
layout: post
title: "RTMFP: the Flash protocol almost nobody implements"
description: "Over UDP, every number a peer sends is an allocation it gets to size. Notes from making fms-m's RTMFP server talk to standard clients and survive hostile ones."
---

Most people who remember Flash video remember RTMP: a TCP connection, chunks of
audio and video, a server in the middle. Fewer remember that Flash Player 10 also
shipped a second protocol, **RTMFP**: UDP-based, encrypted from the first packet,
able to carry many independent streams in one session and to drop late video
instead of stalling on it. It was published as RFC 7016 in 2013.

It is also a protocol almost nobody implements on the server side. It is far
more complex than RTMP, and for most of its life the only client was Flash
Player. [fms-m](https://github.com/damirn/fms-m) does implement it, and most of this
week's work was inside it. Two lessons came out of that.

## Lesson one: Flash-shaped is not spec-shaped

Until May, fms-m's RTMFP worked with exactly one client: Flash Player. That sounds
like it was working. It was not -- it only *looked* like it, because the code had
been written against what Flash happened to do rather than against the protocol.

When I pointed [rtmfp-cpp](https://github.com/zenomt/rtmfp-cpp), written by the
RFC's own author, at the server, it could not even finish the handshake. The
[fix](https://github.com/damirn/fms-m/commit/a260b5c) came down to three
Flash-isms:

- The server advertised Diffie-Hellman groups it did not implement. Flash ignored
  the list; a correct client picked the strongest group offered, sent a 2048-bit
  key, and the server could not use it.
- The client's public key was read from a fixed byte offset in its certificate --
  where Flash happened to put it -- instead of parsing the certificate options.
  The keys never matched, so every data packet failed its checksum and was
  silently dropped.
- Server-to-client streams were tied to a hard-coded flow number that Flash
  tolerated and a correct client rejects.

None of these would ever have shown up in testing against Flash. A second,
independent implementation found all three at once.

## Lesson two: every number is an allocation

RTMFP splits each message into numbered fragments, and the receiver puts them
back in order. Fragments can arrive late or not at all, so the receiver tracks
which sequence numbers are still missing. Every one of those numbers comes from
the peer -- and over UDP, the peer can be anyone.

That turns every number into an allocation the peer gets to size. This week was
mostly about finding each dimension along which that is true:

- **Per jump.** A fragment numbered far ahead of the last one makes the receiver
  record every number in between as missing. With a 64-bit sequence number, one
  packet could ask for billions of entries. A cap on the jump (64K) was
  [added in May](https://github.com/damirn/fms-m/commit/c39151d) -- but 64K
  insertions from a single packet is still a lot, so the
  [missing set itself](https://github.com/damirn/fms-m/commit/8951295) is now
  capped at 8,192.
- **Per fragment.** A flow could buffer at most 8,192 fragments. But a fragment
  can carry up to 64 KiB, so that is half a gigabyte per flow. The bound is now
  [on bytes](https://github.com/damirn/fms-m/commit/4520461): no more than the
  largest message a flow could ever deliver, 16 MiB.
- **Per flow.** One session may open up to 1,024 receiving flows. At 16 MiB each,
  that is 16 GiB. So the flows of one session now
  [share a single budget](https://github.com/damirn/fms-m/commit/c4773d5) of
  32 MiB.
- **Per field.** A "forward sequence number" is the sequence minus an offset, both
  sent by the peer. With the offset larger than the sequence, the subtraction
  wrapped to nearly 2<sup>64</sup>. It is now
  [clamped](https://github.com/damirn/fms-m/commit/8c014bc). And a request to
  abandon fragments up to a number that was never buffered used `find()`, got
  `end()`, and [discarded everything](https://github.com/damirn/fms-m/commit/dce9892).

The pattern is the interesting part. Every bound was correct for the dimension it
named, and every one left the next dimension open: per packet, then per gap, then
per fragment, then per byte, then per flow. An attacker does not respect the
dimension you were thinking about. They multiply along the one you were not.

So the question I now ask about every limit is not "is this bounded?" but
"**what can still multiply it?**"

## Try it

fms-m speaks RTMFP and RTMP to the same streams, so you can publish over one and
play over the other. With rtmfp-cpp's test tools:

```sh
# publish an FLV over RTMFP
tcpublish -4 'rtmfp://127.0.0.1:1935/media#mystream' input.flv

# watch it over plain RTMP
ffplay rtmp://127.0.0.1/media/mystream
```

The server, the tests and all of these fixes are on
[GitHub](https://github.com/damirn/fms-m).
