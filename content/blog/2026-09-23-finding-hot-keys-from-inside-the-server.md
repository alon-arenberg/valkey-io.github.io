+++
title = "The hot key we could not name"
date = 2026-09-23 01:01:01
description = "One shard at 98% CPU, fifteen others bored, and every dashboard insisting the cluster was fine. Finding the key took hours; the tool that would have taken minutes is now in Valkey."
authors = ["alonare"]

[taxonomies]
blog_type = ["Technical Deep Dive"]

[extra]
featured = false
featured_image = "/assets/media/featured/random-03.webp"
+++

The alert is never *"`product:8fd21a` is taking 48,000 requests per second."*

Instead you get one shard in a sixteen-shard cluster pinned at 98% CPU while the other fifteen coast at 30% — and because the dashboards average across all sixteen, the graph everyone watches insists nothing is wrong.

So you add capacity, and nothing improves — which is the moment the diagnosis arrives.
A key lives in one slot, a slot lives on one node, so no number of new nodes can split a single key.
By now you're certain you have a hot key; you just can't name it, and three hours of an incident go into that gap.

*(A composite rather than one night's postmortem, but a shape I've seen more than once.)*

## What we tried

The frustrating part is that Valkey already tells you plenty about your workload — just not this.

Our first instinct was `valkey-cli --hotkeys`, which walks the keyspace asking each key how popular it has been, using the least-frequently-used (LFU) counters.
Those measure long-run popularity with decay — the right input for eviction, and a different question from what is taking traffic *right now*.

`MONITOR` had that truth in real time, but it streams it to a client, so the node we were rescuing does more work while we watch — and we'd still be grepping a firehose for a key we couldn't name.
Slot statistics narrowed it to the hot slot, which felt like progress until we remembered how many keys share one, and client-side sampling would have nailed it had every client been instrumented.

We found the key in the end by correlating that hot slot against a deploy from earlier in the day, which had changed a cache key template.
It worked because someone remembered the deploy, and that's luck wearing a method's clothes.

## What we needed instead

Afterwards, what stood out was how much smaller the question was than the traffic behind it.

That node served around 200,000 requests per second across millions of keys, and all we wanted was a list of ten.
Counting every key to find ten is an expensive way to answer a cheap question, so we wanted a compact list of the heavy hitters, kept by the server and cheap enough to leave running *before* the next incident.

That instinct wasn't new. An earlier attempt by [li-benson](https://github.com/li-benson) in [#2965](https://github.com/valkey-io/valkey/pull/2965) used a Count-Min Sketch, which estimates how often it saw *a key you name* — the wrong half of a problem where naming the key is the problem.

What shipped in [#3708](https://github.com/valkey-io/valkey/pull/3708) takes a different route, Space-Saving, whose appeal is that the ranking falls out of the algorithm rather than bolted on beside it.
Picture sixteen slots, each holding a key name, its database, a count, and an error bound.
On each observed access you increment the key if it holds a slot, take a free one if there is one, and otherwise evict the smallest-count slot and hand the new key that count plus one.

That last step is the whole trick.
Because a new key inherits the score of the one it displaced, it arrives on probation rather than at zero: a genuinely hot key shrugs that off, while a key touched once lands in the weakest slot and is gone by the next access needing the room.
The eviction rule *is* the ranking, so nobody guesses a threshold, and memory stays at a few kilobytes however large the keyspace grows.

## But what does it cost?

Elegant isn't the same as affordable, and a node at 98% CPU is where you can least afford a heavy diagnostic — so we measured it.

We used two `c7g.16xlarge` instances, load generator and server on separate machines: 3 million keys with 512-byte values, a 20% `SET` / 80% `GET` mix over 800 connections, no TLS, no replicas, cluster mode and I/O threads off.
Each point is the mean of five 60-second runs after a 20-second warm-up.
One detail shapes all of it: keys were drawn uniformly at random, so almost every sampled access takes the eviction path and copies a key name — closer to a worst case than a typical one.

![Throughput cost against sampling percentage, by key name length](/assets/media/pictures/hotkeys_sampling_cost.png)

The surprise is the spread between those lines.
Sampling rate alone doesn't set the cost — the length of your key *names* scales it, because that copy on the eviction path gets more expensive the longer the name is.
At the 1% default the cost stays below 0.75% at every key size we tested, inside the benchmark's own run-to-run variation.
Sample every access and it costs about 1.9% with 16-byte names, 3.9% at 128 bytes, 7.1% at 256, and around 12% at 512.
The wiggle in that 16-byte line is noise rather than structure: at short names the effect is close to the 2% that configuration varied between runs.

Tracking more keys, by contrast, is nearly free: at full sampling with 16-byte names, every `hotkeys-top-k` from 1 to 64 landed between 1.2% and 2.1%.
Widen the list rather than the sampling rate.

![GET p99 latency against sampling percentage](/assets/media/pictures/hotkeys_latency.png)

Latency tells the same story from the other side, as it must when the bottleneck is one saturated thread.
With 512-byte names, `GET` p99 drifts from 7.71 ms to 8.78 ms as sampling goes from off to 100%; with 16-byte names it barely moves, 6.93 ms to 7.07 ms.
What matters operationally is that no cliff appears in that sweep, which makes the setting safe to raise on a node already in trouble.

All of which describes this workload on this hardware, with the bias in a known direction: a skewed workload — where a hot key genuinely exists — should cost less, since hot keys stay resident and most accesses become a plain increment. We measured the uniform case.

## The same night, with the tool

Cheap enough to leave on is what changes the night: the answer is already waiting when you go looking.
You notice one node is hot, ask what it's busy with, and read it:

```text
127.0.0.1:6379> CONFIG SET hotkeys-top-k 16
OK
127.0.0.1:6379> HOTKEYS GET
1) 1) "key"
   2) "product:8fd21a"
   3) "db"
   4) (integer) 0
   5) "qps"
   6) (integer) 48200
```

Misses are counted too, which matters more than it sounds: that afternoon's bad template — a key that never existed — shows up here, where a keyspace walk could never find it.
Detection is per-node, so you ask each node and combine the answers yourself.

The limits are worth knowing before you lean on this at 3am.
With no real heavy hitters you still get sixteen entries, because the slots always hold something, and their rates are an artifact of churn rather than your traffic.
The reply gives a rate without the error bound behind it, so read `hotkeys_last_window_samples` in `INFO hotkeys` alongside it. Reads and writes also share one summary today, so a write-hot and a read-hot key look identical.

None of this retires the tools we used that night — `MONITOR` is still right when you need every command, and LFU counters are still the right input for eviction.
What changed is narrow, and it's the part that cost us the hours: getting from "one node is unhappy" to "this key is responsible" no longer depends on someone remembering a deploy.

If the answer isn't the one you needed, [open an issue](https://github.com/valkey-io/valkey/issues) with your version and configuration — knowing which key is hot is only the first question, and I'd like to know what you ask next.
