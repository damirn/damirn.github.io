---
layout: post
title: "Rebuilding a Flash-era media server in C++23"
description: "Modernizing an old server is not about syntax. It is about making every limit explicit, and proving each one."
---

Flash is gone, but its protocols are not. RTMP is still how OBS and ffmpeg push
live video to a server. Apps built for Flash and Adobe AIR still turn up
speaking RTMPE and RTMPT. And RTMFP, Flash's UDP protocol, has almost no open-source server
implementation at all.

[fms-m](https://github.com/damirn/fms-m) is a media server that speaks the whole
family -- RTMP, RTMPE, RTMPS, RTMPT, RTMPTS and RTMFP -- from one binary. Publish
over one, play over any other, record to FLV. It started as an old, Boost-heavy, pre-C++11-style
codebase, and this year I brought it up to C++23.

Here is what I learned: **the syntax was the easy part.**

## The easy part

The core of the mechanical sweep landed in a day. `boost::shared_ptr` became `std::shared_ptr`,
`typedef` became `using`, `NULL` became `nullptr`, `boost::bind` became lambdas,
`boost::thread` became `std::thread`. Clang-tidy did much of the rest.

A few Boost pieces stayed, on purpose. Asio still runs the I/O, one `io_context`
per core. And `boost::tribool` stayed, because the message handlers answer
*yes*, *no*, or *not mine*. On a tribool, `if (r)` is true only for *yes*. On the
"obvious" replacement, `std::optional<bool>`, `if (r)` is true for any answer at
all -- including *no*. A mechanical conversion would have compiled cleanly and
silently inverted the protocol dispatch.

That was the first hint of where the real work was.

## The hard part: limits

A media server takes bytes from strangers. Every count, size and depth in a
protocol is a number an attacker gets to choose, so every one needs a bound.
Most of the audit was finding the places where a bound was *assumed* instead of
*enforced*. Two stories show the shape of it.

**A cap that outlived its reason.** RTMFP numbers every fragment. The server keeps
a running sum of those numbers to detect gaps, and that sum overflows 64 bits
somewhere past 2<sup>32</sup>, so the sequence number needs a cap. At one point the
wire encoding could only carry 29 bits, so the cap
[dropped to 2<sup>29</sup>](https://github.com/damirn/fms-m/commit/1a28776). Later
the encoding was [fixed to follow RFC 7016](https://github.com/damirn/fms-m/commit/1a73956)
-- but the cap stayed at 2<sup>29</sup>, hard-coded. Nothing failed. The diff looked
like a strict improvement. But one busy stream would stop dead after about 11
days at 5 Mbps, every new fragment silently treated as a duplicate. The fix was
[putting the cap back](https://github.com/damirn/fms-m/commit/207f7e3) where the
arithmetic actually needs it.

**A reply that ate events.** Shared objects let clients sync state through the
server, and a reply message can hold at most 64 events. A guard was added:
[stop when the reply is full](https://github.com/damirn/fms-m/commit/5098b76).
Reasonable -- until you notice that joining an object with 62 properties produces
*exactly* 64 events. Anything the client sent after that in the same message was
dropped without a trace: changes never stored, a "leave" that never happened. The
fix was to [start a new reply](https://github.com/damirn/fms-m/commit/4d6145a)
instead of stopping.

Both bugs were introduced by fixes. Both passed every test. That is the pattern.

## The rules that came out of it

Every bug the audit found became a rule, so that the same shape would not recur:

- **Put an invariant in the thing it guards, not in its caller.** A chunk-size
  check lived in the server's connection code, so the client never had it. Moving
  it into the setter fixed both, and made forgetting impossible.
- **A fix covers every twin.** Server and client, RTMP and RTMFP, direct and
  HTTP-tunnelled. Harden one side and the other is the next bug report.
- **Prove a test fails without its fix.** Revert the fix, rebuild, run: the test
  must go red. This caught tests that only checked their own setup -- and a
  sanitizer script that reported a clean run when no video had flowed at all.
- **When a commit changes a limit, say what the old limit bounded.** Then check
  that something still bounds it. The 2<sup>29</sup> cap above is exactly what this
  rule is for.

None of these are new ideas. What surprised me was how often each one was needed,
and how quiet every violation was.

## Try it

```sh
git clone https://github.com/damirn/fms-m && cd fms-m
docker build -t fms-m .
docker run --rm -p 1935:1935 -p 1935:1935/udp -p 8080:8080 fms-m

ffmpeg -re -i input.mp4 -c copy -f flv rtmp://127.0.0.1/media/mystream   # publish
ffplay rtmp://127.0.0.1/media/mystream                                   # watch
```

The code, the tests and the whole history of these fixes are on
[GitHub](https://github.com/damirn/fms-m).
