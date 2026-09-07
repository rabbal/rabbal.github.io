---
title: "Purge at Scale"
date: 2026-09-08 01:10:00 +0330
categories: [Engineering, CDN]
tags: [cdn, caching, purge, performance, ]
description: How CDN purge evolved from Glob/Regex matching in the request hot path to a Trie plus Hash Map design.
---

Caching is one of the core components of a CDN. Keeping content close to users reduces latency, lowers origin load, and increases serving capacity.

But cached content does not always remain valid until its TTL expires. Sometimes content changes earlier, and the cached copy must be removed explicitly. That is **purge**.

At first glance, purge seems simple: receive a URL or pattern, find the corresponding cached content, and remove it. At scale, the choice of **data structure, matching algorithm, and execution location** can directly affect CDN performance.

This post looks at a purge design that started with a generic Glob/Regex approach and evolved into a simpler model based on a **Prefix Tree (Trie)** and a **Hash Map**.

## Where the problem starts

Users can specify an exact path:

```text
/assets/app.js
```

or a pattern:

```text
/assets/*.js
```

The first identifies a single path. The second may cover many objects.

A generic implementation can convert these patterns into Regex expressions and evaluate them against incoming request paths.

Let:

- `R` = requests per second
- `P` = number of purge patterns
- `L` = average path length

If every request checks all patterns, the simplified cost is approximately:

```text
O(R × P × L)
```

When `P` is small, this may be fine. As the number of patterns grows, an operation that looks inexpensive for one request becomes a significant CPU cost at high request rates.

## The first bottleneck: Regex matching

In the initial design, purge patterns were converted into Regex expressions during preparation and used for matching on the request path.

Regex is a powerful, general-purpose tool — a reasonable choice when arbitrary pattern matching is actually required. The problem appears when that matching happens in the **request hot path**.

If every request evaluates multiple Regex expressions, even a small per-match cost is multiplied by requests and patterns. In the best case, a request matches early. In the worst case, no pattern matches and all `P` patterns must be evaluated:

```text
O(P × L)
```

## The cost of Regex compilation and caching

Caching compiled Regex expressions avoids repeated compilation — but those caches have finite capacity. When the number of expressions exceeds effective cache capacity, compilation overhead can reappear.

A simplified model:

```text
C_total ≈ N_requests × N_patterns × C_match + C_compile
```

`C_match` does not have to be large to become expensive. A tiny cost times a large number of requests and patterns is still a real CPU workload.

So the problem is not simply that "Regex is slow." It is the combination of:

- a large number of patterns
- a high request rate
- matching in the hot path
- limited benefit from compiled-expression caching at high pattern counts

## Why moving processing offline was not enough

Moving purge processing into an asynchronous service can reduce edge CPU pressure, but it raises another question: **can we guarantee deterministic invalidation?**

If purge only waits for a future request to refresh content, the old cached copy may still be served until TTL expiry:

```text
P(Cache HIT after purge) > 0
```

A real purge should remove the specified cached content, not merely rely on a future refresh. The new design needed both low CPU overhead and deterministic invalidation.

## Rethinking the data model

Looking at the workload more closely, a large portion of purge operations could be expressed more simply:

> Purge everything whose path starts with this prefix.

For example, `/static/images/*` becomes the prefix `/static/images/`.

That yields a simpler semantic model:

| Pattern | Meaning |
| ------- | ------- |
| `/path/*` | Prefix match |
| `/path/to/file.js` | Exact match |

Instead of a fully generic pattern language, the problem splits into two operations: **exact match** and **prefix match**.

## Option one: linear prefix search

The simplest approach stores prefixes in an array and checks them one by one. Worst-case cost:

```text
T(P, L) = O(P × L)
```

Search time grows linearly with the number of prefixes. In a lab benchmark for this kind of workload, search took about **4.916 seconds**.

## Option two: Prefix Tree (Trie)

A Trie stores strings while sharing common prefixes. For:

```text
/static/images/
/static/icons/
/static/videos/
```

the shared `/static/` is represented once. Lookup follows the request path through the tree, so for path length `L` the search is approximately **O(L)** — no longer linearly dependent on the number of stored prefixes.

## Benchmark results

| Operation | Implementation | Time |
| --------- | -------------- | ---: |
| Search | Linear string array | 4.916 s |
| Search | Trie | 0.001 s |

Relative improvement:

```text
(4.916 - 0.001) / 4.916 × 100 ≈ 99.98%
```

That is the improvement for this **search benchmark** under lab conditions — not a claim of a 99.98% gain in overall CDN performance.

## The cost of building the Trie

A Trie is not free. If `S` is the total length of all prefixes, construction is about `O(S)`. The important distinction is **when** that cost is paid: during preparation, not once per request.

```text
Build once
Search many times
```

> Pay the expensive cost once, not once per request.

## Exact match: where Hash Maps win

For an exact path such as `/static/app.js`, a Trie is unnecessary. Exact lookup fits a Hash Map, with expected **O(1)** lookup.

The architecture uses different structures for different query types:

```text
Exact Path  → Hash Map
Prefix Path → Trie
```

Conceptually:

```text
                Request
                   │
                   ▼
          Exact Match (Hash Map)
                   │
              No Match
                   │
                   ▼
          Prefix Match (Trie)
                   │
              No Match
                   │
                   ▼
             Normal Flow
```

## Exact-match benchmark

| Operation | Structure | Time |
| --------- | --------- | ---: |
| Build | Trie | 12.760 s |
| Build | Hash Map | 0.061 s |
| Search | Trie | 0.264 s |
| Lookup | Hash Map | 0.000265 s |

Lookup-time ratio: `0.264 / 0.000265 ≈ 996` — roughly three orders of magnitude faster for Hash Map in this benchmark.

> The best data structure depends on the query.

## Separating purge data from general configuration

Matching was not the only issue. As purge paths grew, keeping all purge rules inside the main CDN configuration also meant larger config payloads and slower config-related work.

A cleaner approach separates purge rules from general configuration and materializes them into runtime-ready structures during a preparation step:

```text
           Configuration Input
                   │
       ┌───────────┴───────────┐
       ▼                       ▼
General CDN Config        Purge Rules
                               │
                               ▼
                 Preparation / Materialization
                               │
                               ▼
                    Prepared Purge State
                               │
                       ┌───────┴───────┐
                       ▼               ▼
                   Hash Map          Trie
```

That separates **configuration distribution** from **request-time execution**.

## Why Purge All is an interesting edge case

`Purge All` can look like `path = /*` and `host = *.example.com`. If normal matching assumes a concrete hostname, a wildcard host may not behave as intended. Operations like Purge All need explicit semantics — and sometimes dedicated handling.

> Define the semantics of an operation before optimizing its implementation.

## Purge and HTTP Vary

`Vary: Accept-Encoding` means one URL can map to multiple cached representations:

```text
URL ≠ Cache Object
CacheKey = f(URL, VaryHeaders)
```

Deleting one object does not necessarily delete every variant for that URL. Correct purge must account for `Vary` metadata, which adds I/O and processing cost. A multi-level cache can reduce lookups, but the trade-off remains:

```text
Correctness ↔ I/O Cost
```

Purge is not only string matching. It can involve URL, headers, and content negotiation.

## Can active purge be simulated?

For large numbers of exact paths, batching probe requests to cache nodes and verifying cache state can work for some workloads. `Vary` limits this: one request does not cover every representation of a URL. Treat that approach as an **operational workaround**, not a general replacement for purge semantics.

## The main design result

The redesign was not just a code optimization. The deeper issue was using a highly generic abstraction:

```text
Glob → Regex → Matching
```

for a workload that often reduced to:

```text
Exact Match + Prefix Match
```

The resulting flow:

```text
         Purge Request
               │
               ▼
          Preparation
               │
       ┌───────┴───────┐
       ▼               ▼
    Hash Map         Trie
       │               │
       └───────┬───────┘
               ▼
        Prepared State
               │
               ▼
         Request Path
               │
       ┌───────┴───────┐
       ▼               ▼
  Exact Match     Prefix Match
       │               │
       └───────┬───────┘
               ▼
             Purge
```

CPU-heavy work moved out of the repetitive request path into preparation:

> Do expensive work once, not once per request.

## Engineering lessons

**1. Complexity is practical.** `O(P × L)` vs `O(L)` may not matter when `P` is small. At scale, it affects CPU and latency.

**2. Regex cost is not just one match.** In the hot path, evaluate:

```text
Total Cost ≈ Requests × Patterns × Match Cost + Compile Cost
```

**3. Limiting capabilities can be an optimization.** If the real workload is mostly prefix-based, Glob/Regex flexibility may not justify its runtime cost. Sometimes the best optimization is removing an abstraction you do not need.

**4. Separate preparation from runtime.** If `C_total = C_p + N × C_r` and you can make `C_r' << C_r`, the benefit grows with `N`.

**5. Choose data structures based on the query model.**

| Query type | Suitable structure |
| ---------- | ------------------ |
| Exact lookup | Hash Map |
| Prefix lookup | Trie |
| Arbitrary pattern matching | Regex / Glob engine |

## A note on benchmarking

Benchmarks without context mislead. Results depend on language, CPU, allocator, string lengths, entry counts, distribution, access patterns, locality, and implementation details. `0.001 s` is not a universal guarantee. Relative improvement under the same environment and workload is what matters.

## Conclusion

The core problem with purge at scale was not merely deleting cached content. It was placing a relatively expensive operation in the **request hot path**.

The redesign focused on three changes:

1. Remove Glob/Regex from the primary matching path.
2. Use a **Trie** for prefix matching.
3. Use a **Hash Map** for exact matching.

In high-scale systems, ask not only "how fast is this operation?" but also "how many times is it executed, where, and on how much data?"

> Measure the workload, simplify the problem, choose the right data structure, and move expensive work out of the hot path.
