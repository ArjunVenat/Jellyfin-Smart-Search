# ADR-002: Separate Throughput (Offline Indexing) from Latency (Query Serving)

- **Status:** Proposed
- **Date opened:** 2026-11-23
- **Date decided:**
- **D15 row:** 3

## Context

The system has two workloads with different goals:

- **Throughput path: offline batch indexing of the media library.** Decode video, detect shots, extract frames and subtitles, embed, store. Measured in video hours per day and GPU-seconds per video hour. Latency per item barely matters; total completion time and cost do.
- **Latency path: one user query.** Embed the query, search, fuse, return. Measured in p50/p95 milliseconds. Volume is tiny, but every query is waited on.

*Designing Data-Intensive Applications*, *AI Engineering* and *Inference Engineering* all argue for optimizing these two separately: batch systems trade latency for throughput (large batches, queues, retries), while serving systems do the opposite.

Hardware constraints (D2): one RTX 4060 Laptop GPU (8 GB VRAM) and CPU-only mini PCs.

<!-- Things this ADR has to settle:
- Hardware: does the query path share the GPU laptop with batch indexing, or run elsewhere (e.g. CPU on a mini PC)?
- Contention: if they share the GPU, how do queries avoid waiting behind batch jobs (priority, pausing, scheduling)?
- Availability: what happens to search when the laptop is asleep or away?
- Model coupling: query embeddings must come from the SAME model and version as the indexed embeddings, or the vectors aren't comparable. The two paths can use different hardware but not different models.
- Background research: real-world examples of offline batch media processing (see ../notes.md).
-->

## Options Considered

| Option | Pros | Cons | Cost (time / GPU / complexity) |
|---|---|---|---|
| | | | |

## Decision Rule
<!-- Write this BEFORE measuring, e.g. a target p95 query latency while indexing is running, and a target video hours/day. -->

## Experiment and Evidence

## Decision

## Consequences
