---
layout: post
title: "Predication in Elastic CGRAs"
date: 2026-02-01 09:00:00-0500
description: Extending an elastic CGRA and its CAD flow with partial and full predication, via a local-predicate design and a dedicated global predicate network.
tags: CGRA architecture predication dataflow CAD EDA
categories: research
giscus_comments: false
related_posts: false
inline: false
toc:
  sidebar: left
---

Published at **CGRA4HPC @ HPCA 2026**, New Orleans, USA, 2026.

**Omkar Bhilare**, Omar Ragheb, Boma Adhi, Kentaro Sano, Jason Anderson, Tomohiro Ueno
*(University of Toronto · Fujitsu Consulting Canada · RIKEN-CCS)*

<div class="row mt-3 mb-4">
  <div class="col-6">
    <a class="btn btn-outline-primary w-100" href="https://drive.google.com/file/d/1pgjqz9EvrQgBwBdMbz7SaWBP2fGKBGOg/view" role="button" target="_blank">Paper</a>
  </div>
  <div class="col-6">
    <a class="btn btn-outline-primary w-100" href="https://utoronto-my.sharepoint.com/:p:/g/personal/omkar_bhilare_mail_utoronto_ca/IQAVdPrt5naiTo9TvRpkLHtAAQI_Q8aa2v--VLx_PpOdTiI?e=Fmbtyn" role="button" target="_blank">Slides</a>
  </div>
</div>

## Background

Coarse-grained reconfigurable arrays (CGRAs) accelerate compute-intensive
kernels using a 2D grid of word-level processing elements (PEs) and a
programmable interconnect. *Elastic* (latency-insensitive) CGRAs follow the
dataflow paradigm: compute and communication fire when data arrives, signalled
by `valid`/`stop` handshaking that conveys data readiness and back-pressure.
This makes them well suited to kernels whose latencies are not known at compile
time, such as those gated by external memory.

The catch is control flow. A loop with no conditionals maps cleanly to a CGRA,
but most real loops are not like that — a classic study found that 56–84% of
executed loops contain conditional branches. The standard way to express such
control flow on spatial hardware is **predication**: associating a TRUE/FALSE
predicate with an event so that the predicate decides whether (or where) data
moves. This work extends the RIKEN elastic CGRA and the open-source CGRA-ME 2.0
CAD flow with predication support. A key subtlety is that, in an elastic
fabric, the predicate signals themselves need handshaking so they stay aligned
with the events they gate.

## Two predication schemes

Consider `z = <cond> ? a + b : a - b`. There are two ways to realize it:

- **Partial predication** computes *both* the sum and the difference, then uses
  a **select** to forward the correct result based on the predicate. The
  hardware is simple — essentially one select multiplexer — but the array does
  speculative, ultimately unnecessary, work.
- **Full predication** executes *only* the chosen path. A **branch** primitive
  steers data down one path and invalidates the other, and a **merge** primitive
  recombines the live tokens. It avoids wasted work (a likely power, and
  possibly performance, win) but is harder to map, since branch and merge often
  need to live in separate switch blocks.

A small set of new elastic primitives supports both schemes: a *truncate* unit
(32-bit condition → 1-bit predicate), 1-bit I/O, a *select* for partial
predication, and a *fork-branch* plus *merge* for full predication. Crucially,
these are folded into the switch blocks (SBs) by reusing the existing crossbar
forks and multiplexers, rather than adding logic inside the PEs — which keeps
the area cost low and frees PEs for real computation.

## Two architectures

We explore two ways to move predicates around the array.

{% include figure.liquid loading="eager" path="assets/img/cgra4hpca/architectures.png" class="img-fluid rounded z-depth-1" zoomable=true caption="Local predication (left): predicates are produced and consumed within the same tile over the 32-bit fabric. Global predicate network (right): a dedicated 1-bit routing network with its own 1-bit switch blocks, ALU, and perimeter I/Os." %}

- **Local predication.** The PE's ALU performs the compare to produce a 1-bit
  predicate that is consumed in the *same* tile. The interconnect stays 32-bit
  wide, as in the baseline. The cost is a placement constraint: the predicate
  producer and consumer must be co-located, which creates mapping bottlenecks,
  especially when one predicate fans out to many consumers.
- **Global predicate network.** A separate, dedicated 1-bit interconnect routes
  predicates anywhere on the array, with its own 1-bit switch blocks
  (orthogonal connections only), a 1-bit ALU for logical ops (AND/OR/XOR), and
  1-bit perimeter I/Os. Predicates no longer need to be consumed locally, which
  relaxes placement and improves resource utilization.

## Key contributions

- A predication-enabled extension to an elastic CGRA and the CGRA-ME 2.0 CAD
  flow, with handshaking-aware predicate signals.
- Switch-block primitives for both partial and full predication that **reuse
  existing crossbar resources** for low overhead.
- Two architectures — local predicate and global predicate network — with a
  thorough area, performance, and mappability comparison across grid sizes.

## Results

We mapped a 20-benchmark suite of synthetic control-flow DFGs (10 each for full
and partial predication) using CLUMAP in CGRA-ME 2.0, and pushed five grid sizes
(2×2 to 10×10) through Synopsys Design Compiler, Cadence Innovus, and PrimeTime
on the Nangate 45nm library, over-constrained to a 2 GHz target.

- **Mappability strongly favors the global network.** For full predication on a
  4×4, the local variant maps only 6/10 benchmarks, versus 9/10 (seven at 100%)
  for the global network. Scaling to 8×8 barely helps local predication, while
  the global network maps every benchmark. The recurring failure mode for local
  predication is the producer→consumer co-location constraint, which collapses
  under predicate fanout.
- **Better PE utilization.** A 5-node example (two 32-bit compares + three 1-bit
  logic ops) on a 2×2 over-subscribes the 32-bit PEs locally (unmappable), but
  the global network offloads the 1-bit logic onto dedicated 1-bit PEs and maps
  successfully.
- **Modest, well-bounded area cost.** Thanks to crossbar reuse, local-predicate
  overhead is ~6–10% over the baseline, and the global predicate network is
  ~12–20%. Delay is essentially unchanged on average — reusing the crossbar does
  not lengthen the critical path — with small swings attributable to EDA noise.

The headline trade-off: the global predicate network buys substantially higher
mapping success for a modest additional area cost, while local predication is
the cheapest option but is constrained in what it can map.

## Conclusion

Predication brings conditional execution to elastic CGRAs, broadening them
beyond feed-forward streaming kernels to loops with real control flow. We
studied partial and full predication on two architectures — one where predicates
are produced and consumed locally, and one with an independent 1-bit predicate
routing network. The global predicate network achieves markedly higher
mappability at a higher, but still modest, silicon cost. As future work, we plan
a comparative power analysis of full vs. partial predication across both
architectures.
