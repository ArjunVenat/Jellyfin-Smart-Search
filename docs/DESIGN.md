# Jellyfin Smart Search: Design Doc

**Stage:** v0. Written before any code. Open questions are listed in D15.
<!-- v1: updated with baseline numbers. v2: final, with results. Update this line and the Changelog at each stage. -->

**Related docs:**
- [notes.md](notes.md): research notes and brainstorming. Not design.
- [arch/](arch/): Architecture Decision Records (ADRs), one per decision. Template: [ADR-000-template.md](arch/ADR-000-template.md).

<!--
How to use this doc:
- DESIGN.md says WHAT the system must do, HOW success is measured, and WHICH decisions are still open.
- When a decision is made, write an ADR in arch/ and link it from the D15 table. Don't argue the choice here.
- Anything that is "this technique does X" (not "this system needs X") goes in notes.md.
-->

---

## D1. Context and Problem

Jellyfin is a free and open source media server for sharing digital media files. It is analogous to a streaming service such as Netflix, but suited to a local media library. Users can import files they own into the media library.

<!-- TODO: What's broken today? (Jellyfin search covers titles and metadata only, not moments inside a video.) Who has the problem, and why is a timestamp worth more than a title? -->

## D2. Constraints

**Hardware**
- **GPU:** one laptop with an NVIDIA RTX 4060 Laptop GPU (8 GB VRAM). This is the only GPU, and the main candidate for embedding work, to avoid cloud costs.
  - 8 GB VRAM limits which model sizes and batch sizes fit.
  - A laptop can sleep, overheat and slow down, or leave the house.
- **Mini PCs:** CPU-only. Candidates for decoding video and streaming frames to the GPU laptop.
- **Storage:** media on TrueNAS, reached over NFS (see D6).

<!-- TODO: time budget, privacy (what may and may not leave the homelab), legal or ethical limits (e.g. no face recognition), own library only. -->

## D3. Goals and Non-Goals

### Goals

When a user of the Jellyfin server searches for something like "red car" or "man and woman dancing", the goal is for Jellyfin to return the movies or TV shows most similar to that search, at the exact timestamp. For example, the "man and woman dancing" query should bring up scenes like Mia and Vincent dancing in Jack Rabbit Slim's or Mario and Gloria dancing in 8½, provided both movies are in the Jellyfin server's library. The media items that come up should be shown with the progress bar at the timestamp for the specific scene. This is similar to YouTube, where clicking a partially watched video, which shows a partial progress bar, starts playback at that timestamp.

<!-- TODO: turn the paragraph above into a short list of goals. -->

### Non-Goals

<!-- TODO: what you are explicitly NOT doing, e.g. fine-grained motion? named characters? other clients besides web? multi-language? -->

## D4. Success Metrics

<!-- Name the metrics that decide things. Targets can say "TBD after baseline" at v0. -->

| Metric | Measured on | Target | Baseline |
|---|---|---|---|
| | | TBD | TBD |

## D5. Evaluation Plan

<!-- TODO:
- What counts as a hit?
- Query types (taxonomy)
- Queries with one answer vs. many answers
- Who writes the queries, and how bias is limited
- Dev/test split, and when the test set is frozen
- Eval corpus (the titles indexed for experiments) vs. the full library
- Baselines every option is compared against
- How a difference between two options is judged significant
-->

## D6. Data Sources and Scope

The media lives on a TrueNAS server on the LAN. Nginx Proxy Manager is set up on the NAS so that a hostname can be used to reach the server. The files themselves can be accessed through an NFS share. Processing media files on the TrueNAS server is not ideal; a more powerful device that supports GPU passthrough should do it. The server that is optimized for streaming media through Jellyfin can be used the same way, streaming keyframes to a device with a GPU for embedding. The mini PCs are another candidate for decoding and streaming frames (see D2).

<!-- TODO: library size (titles, hours), subtitle coverage (text vs. image-based, external vs. embedded), titles with no subtitles. -->

## D7. System Overview

<!-- High-level flow and an architecture diagram. At v0, describe candidate shapes and point to the open decisions in D15. -->

Two workloads with different goals (see [ADR-002](arch/ADR-002-throughput-vs-latency.md)):
- **Offline indexing (throughput):** batch processing of the media library.
- **Query serving (latency):** answering one user search.

## D8. Data Model

<!-- Tables/entities, keys, and versioning (model and pipeline version on derived data). -->

## D9. Ingestion

<!-- Getting files to the workers: new, changed or deleted items, the queue, idempotency, recovery. -->

## D10. Extraction

<!-- Shots/segments, keyframes, subtitles, captions, embeddings. Known failure cases. -->

## D11. Retrieval and Ranking

<!-- Indexes, query flow, fusion, merging duplicate results, optional re-ranking or verification. -->

## D12. Agent and Tools

<!-- If it's built: what job the agent has that a pipeline can't do, its tools, its budgets, and how it's compared fairly against the pipeline. -->

## D13. Safety, Privacy, and Security

<!-- Per-user access control, API token scope, prompt injection, confirmation before actions, what data may leave the homelab. -->

## D14. Observability, Cost, and Latency

<!-- Tracing, GPU-seconds per video hour and per query, latency budget per stage, CI eval. -->

## D15. Open Decisions

<!-- One row per decision. Write the decision rule BEFORE running the experiment. Link the ADR once it exists. -->

| # | Decision | Options | Decided by (metric + rule) | Depends on | Decide by | Status | ADR |
|---|---|---|---|---|---|---|---|
| 1 | Integration approach (e.g. fork behind a feature flag, sidecar, ...) | | | | | Open | |
| 2 | Overall retrieval approach | | | | | Open | [ADR-001](arch/ADR-001-multimodal-rag-pipeline.md) |
| 3 | Separate the throughput path (offline indexing) from the latency path (per-query serving) | | | | | Open | [ADR-002](arch/ADR-002-throughput-vs-latency.md) |

## D16. Risks and Milestones

### Risks

| Risk | Impact | Mitigation / planned experiment |
|---|---|---|
| Capturing movement while indexing frame by frame will be a challenge. | | |
| Only one GPU, and it's a laptop (8 GB VRAM): limits model size and indexing throughput, and the laptop may not always be available. | | |

### Milestones

| Week | Milestone | Done when |
|---|---|---|
| | | |

---

## Changelog

| Date | Stage | Change |
|---|---|---|
| 2026-11-23 | v0 | Restructured into D1–D16; brainstorm moved to notes.md; hardware constraints added; ADR-001 and ADR-002 opened. |
