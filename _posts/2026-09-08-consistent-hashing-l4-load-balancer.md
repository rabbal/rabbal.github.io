---
title: "Stop Reshuffling Connections: Consistent Hashing in an L4 Load Balancer"
date: 2026-09-08 01:30:00 +0330
categories: [Engineering, Networking]
tags: [load-balancing, consistent-hashing, ebpf, networking, tcp, performance]
description: Why an eBPF L4 load balancer needs consistent hashing — and how a control plane can compile a hash ring into a fixed map so backend changes move as little traffic as possible.
---

A load balancer should be allowed to change its mind about a backend.

It should **not** be allowed to change its mind about every other backend at the same time.

That distinction matters enormously for Layer 4 load balancing — especially when the data plane is an **eBPF** program running in the kernel, where every extra instruction and branch sits on the packet path.

When a backend becomes unhealthy, adding or removing it from the pool is expected. What is not expected is for thousands or millions of otherwise healthy connections to suddenly map to different destinations simply because one backend disappeared.

That is exactly what can happen with a conventional weighted lookup table.

The solution is **consistent hashing**.

There is an interesting constraint: the eBPF data plane does not perform a traditional consistent-hash lookup. It uses a small, fixed-size destination table — typically an eBPF map.

So instead of making the eBPF program more complicated, the complexity moves into the control plane. Consistent hashing is used to **compile a dynamic hash ring into a fixed lookup table**.

That gives the property that actually matters:

> When the backend pool changes, move as little traffic as possible.

## The problem with "just rebuild the table"

Start with a simple load balancer. Suppose there are 100 destination slots and four backends:

```text
A A A A A A A A A A
A A A A A A A A A A
A A A A A A A A A A
A A A A A A A A A A
B B B B B B B B B B
B B B B B B B B B B
C C C C C C C C C C
C C C C C C C C C C
C C C C C C C C C C
D D D D D D D D D D
```

The exact distribution can be weighted, but the idea is the same: each backend owns a contiguous range of slots.

Now imagine B becomes unhealthy. The obvious implementation is:

1. Remove B.
2. Recalculate the weights.
3. Rebuild the entire table.

It sounds harmless. It isn't.

The problem is that the meaning of a slot changes. A client that previously hashed to slot 42 might have been sent to A. After rebuilding, slot 42 might belong to C.

Nothing happened to A. Nothing happened to that client. Yet the mapping changed anyway.

For a stateless HTTP request, this may be barely noticeable. For an established TCP connection, it can be disastrous.

## TCP doesn't care that your table changed

An L4 load balancer sits below the application layer. It does not get to say:

> This request can safely be retried against another server.

An established TCP connection has state. If traffic that previously went to Backend A is suddenly sent to Backend C, the new backend may have no knowledge of that connection:

```text
         Client
            │
            │ existing TCP connection
            ▼
      Load Balancer
            │
    ┌───────┴───────┐
    ▼               ▼
 before           after
Backend A       Backend C
```

The backend change was legitimate. The collateral damage was not.

That is the real reason consistent hashing matters in an L4 load balancer. It is not primarily about a mathematically elegant distribution. It is about **preserving stability**.

## Make the eBPF data plane boring

The easiest fix would be a full consistent-hash lookup in the packet path: walk a ring, binary-search tokens, apply weights on every packet. This design deliberately does not do that.

An eBPF L4 load balancer is a strong reason not to. Programs attached at XDP or TC run early in the kernel networking stack. They are verifier-constrained, latency-sensitive, and a poor place for dynamic ring construction or open-ended search. What they are good at is a short, predictable path over maps.

So the eBPF data plane stays deliberately boring:

```text
  client IP
      │
      ▼
    hash
      │
      ▼
 slot index
      │
      ▼
eBPF map (destination table)
      │
      ▼
   backend
```

The destination table is a fixed-size map. The eBPF program does not know about virtual nodes, rings, backend weights, token positions, or binary searches. That is a feature.

The userspace control plane does the complicated work and produces a simple artifact the program can consume:

```text
destinations[0..99]
```

When backends change, the control plane rebuilds the map contents. The eBPF program keeps doing the same cheap lookup.

> Build complexity once in the control plane so that the eBPF data plane can remain extremely cheap.

## A consistent hash ring behind a fixed table

The control plane constructs a conventional virtual-node consistent hash ring. For every backend, it generates multiple virtual nodes.

If:

```text
virtual nodes per weight unit = 150
```

then:

```text
weight 1 → 150 virtual nodes
weight 2 → 300 virtual nodes
weight 4 → 600 virtual nodes
```

A backend's weight is represented by the number of tokens it contributes to the ring. Each token gets a deterministic position:

```text
position = xxHash64(backend identity || replica number)
```

Conceptually:

```text
              A
        ●           ●
     ●                 ●
   C                     B
     ●                 ●
        ●           ●
              C
```

Once all tokens are generated, they are sorted by 64-bit position. That is the ring.

## Why the hash function matters

This is one of those details that looks unimportant until it breaks assumptions.

You might think any fast hash is good enough. Not necessarily.

Virtual-node positions need to behave like independent random points on the ring. Consider two similar backend addresses:

```text
10.0.0.1
10.0.0.2
```

A weakly mixing hash can preserve correlations between these inputs. Virtual nodes for the two backends can become correlated too. With a small number of slots, those correlations become very visible.

In extreme cases, two equally weighted backends can produce something like:

```text
96 slots → A
 4 slots → B
```

instead of:

```text
50 slots → A
50 slots → B
```

That is not a theoretical curiosity. When the output space is small, bad statistical properties become operational problems.

**xxHash64** provides strong avalanche behavior while remaining extremely fast. Small input changes should produce thoroughly different outputs — so `hash(A, replica)` and `hash(B, replica)` behave as unrelated positions even when A and B look almost identical.

## The hash output is the coordinate

Another design choice: use the complete 64-bit hash output as the ring coordinate.

The ring spans `0 … 2⁶⁴ − 1`. There is no `hash % ringSize` and no truncation to 32 bits. The hash itself is the coordinate.

That gives an enormous address space:

```text
2⁶⁴ ≈ 1.84 × 10¹⁹
```

With only thousands of virtual nodes, collisions are extraordinarily unlikely. More importantly, the model stays clean:

```text
backend + replica
        │
        ▼
    xxHash64
        │
        ▼
  ring coordinate
```

## Bridging thousands of tokens to 100 slots

There may be thousands of points on the ring, but the data plane only has a fixed number of physical destination slots. Those worlds need a bridge.

Give every physical slot a deterministic position on the same ring. For slot `i`:

```text
slotPosition(i) = xxHash64(i)
```

Then perform the standard consistent-hashing lookup:

```text
slot position
      │
      ▼
first token clockwise
      │
      ▼
backend
```

If the search reaches the end of the ring, it wraps to the first token.

The result is written into the physical destination table (the eBPF map). The program still does nothing more complicated than:

```text
client hash → slot → backend
```

## Why hash the slots too?

Slots could be placed at evenly spaced positions. They are not:

```text
slotPosition(i) = xxHash64(i)
```

That spreads probes pseudo-randomly through the same 64-bit space, so the physical table structure does not introduce another deterministic pattern.

The two hashing operations are independent:

```text
Backend + replica          Slot number
        │                       │
        ▼                       ▼
    xxHash64                xxHash64
        │                       │
        ▼                       ▼
 Virtual-node              Slot position
   position
```

Both live in the same coordinate system.

## The payoff: removing a backend

Suppose A, B, and C have equal weights, and B becomes unhealthy.

With a sequential table, removing B can shift the boundaries of A and C.

With consistent hashing, remove B's virtual nodes from the ring. That is it.

Slots that previously resolved to B now resolve to the next available token. Slots that resolved to A stay with A. Slots that resolved to C stay with C:

```text
Before:

A ─── B ─── C ─── A ─── B ─── C

After removing B:

A ───────── C ─── A ───────── C
```

Some slots must move — traffic that belonged to B has to go somewhere. The important part is **who does not move**. A does not move because B disappeared. C does not move because B disappeared. Only B's portion of the key space needs reassignment.

## Close to the minimum possible movement

For `N` equally weighted backends, removing one means approximately `1 / N` of the key space must move:

| Backends | Approx. affected key space |
| -------: | -------------------------: |
|        2 |                        50% |
|        3 |                        33% |
|        4 |                        25% |
|       10 |                        10% |

If one out of ten backends disappears, there is no good reason for the other nine to exchange traffic among themselves. Consistent hashing encodes that intuition in the algorithm.

## Adding a backend is local too

The same principle applies in the other direction. Given A and B, adding C introduces new virtual nodes that take ownership of portions previously owned by A and B.

Expect:

```text
A → A or C
B → B or C
```

Not:

```text
A → B
B → A
```

just because C was added.

> A new backend should primarily steal traffic, not cause unrelated traffic to move sideways.

## Weighted backends fall out naturally

If one backend has twice the capacity of another, give it approximately twice as many virtual nodes:

```text
A: weight 4 → 600 tokens
B: weight 2 → 300 tokens
C: weight 3 → 450 tokens
D: weight 1 → 150 tokens
```

Expected ownership is roughly:

```text
A → 40%
B → 20%
C → 30%
D → 10%
```

The ring provides proportional distribution. The fixed destination table provides the mapping consumed by the data plane.

## Catch: 100 slots are still 100 slots

Consistent hashing does not create more resolution than the data plane provides.

With 100 physical slots and 50 equal-weight backends, the theoretical average is 2 slots per backend. A backend cannot own 2.37 slots — slots are discrete.

Distribution accuracy eventually becomes dominated by table size rather than hash quality. More virtual nodes improve the statistical quality of the ring. They do **not** increase the resolution of a 100-entry destination table.

If finer-grained distribution across many backends is needed, more physical slots are ultimately required.

## Weight changes are more expensive

Consistent hashing gives excellent locality for backend additions and removals. Weight changes are different.

Changing A from weight 2 to weight 4 is not just metadata — it changes A's token population. That introduces many new virtual nodes and remaps ownership across parts of the ring. A weight change can cause substantially more remapping than adding or removing a backend.

Membership changes have a strong locality guarantee. Not every configuration change has the same disruption characteristics.

## Consistent hashing and connection affinity

Connection affinity benefits from this design. Imagine the data plane caches:

```text
client IP → slot 42
```

As long as slot 42 points to the same backend, that cached decision remains useful.

With a globally reshuffled table, the cache can go stale without the client changing:

```text
client → slot 42 → Backend A
                    │
                 rebuild
                    │
                    ▼
client → slot 42 → Backend C
```

The cache still says "slot 42," but slot 42 now means something else.

Consistent hashing reduces this because most slots stay associated with the same backend after an unrelated backend change:

```text
Consistent hashing
        │
        ▼
 fewer slot changes
        │
        ▼
 more stable affinity
        │
        ▼
fewer unnecessary connection moves
```

## No need to persist the ring

Ring construction can stay stateless. The control plane already has authoritative backend state. Whenever it needs to synchronize the destination table, it reconstructs the ring:

```text
backend state
      │
      ▼
generate tokens
      │
      ▼
 sort tokens
      │
      ▼
  map slots
      │
      ▼
destination table
```

There is no need to maintain a persistent ring and mutate it incrementally.

> The destination table is derived state.

Given backends, weights, hash algorithm, and virtual-node configuration, the same table can be reproduced. That keeps the implementation deterministic and easier to reason about.

## Complexity lives where it belongs

Building the ring requires sorting virtual nodes: `O(N log N)`, where `N` is the number of virtual nodes.

Each physical slot then does a binary search: `O(log N)`. Mapping all slots costs `O(S log N)`, where `S` is the number of physical slots.

For a small fixed table, that is cheap — and it happens in the control plane. The eBPF packet path stays simple.

> Spend CPU during configuration changes to save complexity and latency on every packet.

## What to test

A consistent-hashing implementation should be tested for properties, not just code coverage:

- **Determinism** — same backend state produces the same mapping.
- **Single backend** — with one healthy backend, every slot resolves to it.
- **Zero weight** — a backend with zero effective weight owns no ring space.
- **Proportional weights** — a 2:1 weight ratio yields roughly a 2:1 slot ratio, within table limits.
- **Backend removal** — unaffected backends retain their existing slots.
- **Backend addition** — the new backend primarily takes ownership rather than causing existing backends to exchange ownership.
- **Token determinism** — the same backend and replica always produce the same ring position.
- **Token distribution** — collisions and pathological clustering should be extremely rare.

## The bigger lesson

It is tempting to treat load balancing as a problem of distributing traffic evenly. That is only half the problem.

In a real L4 system, **stability is just as important as distribution**.

A theoretically perfect distribution that constantly moves existing connections is not necessarily a good load balancer. A slightly less perfect distribution that preserves established mappings during topology changes can be much more useful operationally.

The design optimizes for both:

```text
                 Load Balancer
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
     Distribution             Stability
          │                       │
    virtual nodes          consistent hashing
          │                       │
          └───────────┬───────────┘
                      ▼
              fixed destination
                 eBPF map
                      │
                      ▼
               simple eBPF
                data plane
```

The ring handles the dynamic world of changing backends. The fixed map keeps the eBPF packet path simple. Together they give something more valuable than either technique alone: a load balancer that changes when it needs to, without constantly forgetting where everything else was.

## Final takeaway

Backend failures, additions, and capacity changes are inevitable. A full reshuffle every time they happen does not have to be.

A consistent hash ring turns a global operation into a local one:

```text
Backend changes
      │
      ▼
Ring changes locally
      │
      ▼
Only affected slots move
      │
      ▼
Most affinity mappings survive
      │
      ▼
Fewer unnecessary TCP disruptions
```

The key design principle is simple:

> When one backend changes, don't make the entire system change with it.

That is the real value of consistent hashing in an L4 load balancer.
