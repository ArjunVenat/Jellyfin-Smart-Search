# Prior art: batch media processing and in-video search

Resolves issue #4. Feeds [ADR-002](../arch/ADR-002-throughput-vs-latency.md) and the open search-unit decision (see [notes.md](../notes.md)). Researched 2026-11-26.

Terms follow `GLOSSARY.md`: a **Moment** is a result, a **Media item** is one movie or episode, a **Segment** is the indexed unit, and **Shot** and **Scene** keep their film meanings.

## Sources and lead check

All five leads in `notes.md` exist. The Netflix posts are on Medium (`netflixtechblog.com`), which returned HTTP 403 to automated fetches. I read them through a verbatim syndicated mirror (`noise.getoto.net`) and cite both URLs. Quotes below come from that mirror.

| Lead in notes.md | Exists? | Source |
|---|---|---|
| "Building In-Video Search" | Yes. Netflix, 2023-11-06, Boris Chen, Ben Klein, Jason Ge, Avneesh Saluja et al. | [original](https://netflixtechblog.com/building-in-video-search-936766f0017c), [mirror](https://noise.getoto.net/2023/11/06/building-in-video-search/) |
| "Match Cutting at Netflix" | Yes. Full title "Match Cutting at Netflix: Finding Cuts with Smooth Visual Transitions", 2022-11-17. Paper at WACV 2023. | [mirror](https://noise.getoto.net/2022/11/17/match-cutting-at-netflix-finding-cuts-with-smooth-visual-transitions/), [paper](https://arxiv.org/abs/2210.05766) |
| "Scaling Media Machine Learning at Netflix" | Yes. 2023-02-13, Gustavo Carmo et al. | [mirror](https://noise.getoto.net/2023/02/13/scaling-media-machine-learning-at-netflix/) |
| "the Cosmos platform" | Yes. "The Netflix Cosmos Platform", 2021-03-01, Frank San Miguel. **The notes call it "queued, resumable workflows"; the post says queued, but never uses "resumable", "retry" or "idempotent".** | [mirror](https://noise.getoto.net/2021/03/01/the-netflix-cosmos-platform/) |
| Uber Michelangelo | Yes. "Meet Michelangelo: Uber's Machine Learning Platform", 2017-09-05, Jeremy Hermann and Mike Del Balso. | [uber.com](https://www.uber.com/en-AU/blog/michelangelo-machine-learning-platform/) |

Extra sources I added because they answer the ticket's questions better:

- Netflix, "Building a Media Understanding Platform for ML Innovations" (2023-03-14): the serving side of in-video search. [mirror](https://noise.getoto.net/2023/03/14/building-a-media-understanding-platform-for-ml-innovations/)
- Netflix, "Scalable Annotation Service — Marken" (2023-01-26): how derived data is stored and versioned. [mirror](https://noise.getoto.net/2023/01/26/scalable-annotation-service-marken/)
- CLIP4Clip (Luo et al., arXiv 2104.08860): mean pooling vs. temporal modelling for CLIP-based video retrieval. [arXiv](https://arxiv.org/html/2104.08860v2)
- Immich, the self-hosted photo server with CLIP smart search. It is the closest homelab-scale prior art: one box, background jobs, a swappable embedding model. [Searching docs](https://docs.immich.app/features/searching/), [Remote ML guide](https://docs.immich.app/guides/remote-machine-learning/), [issue #32726](https://github.com/immich-app/immich/issues/32726), [discussion #10650](https://github.com/immich-app/immich/discussions/10650)

## 1. Search unit and motion in Netflix in-video search

- **Segment = Shot.** "Video embeddings are computed per shot" ... "stored in the media feature store and replicated to an Elasticsearch cluster for real-time nearest neighbor queries." The query side: "the text encoder from the same trained model" turns the query into an embedding, which is matched against the shot embeddings. A user can search a subset of titles or the whole catalog.
- **Motion is handled by pooling over the shot plus fine-tuning on video, not by a separate motion model.** The post starts from pretrained image-text weights and fine-tunes on internal shot–text pairs. For short units it notes "an unparameterized aggregation like averaging (mean-pooling) can be more effective". It reports that moving to video "provided a substantial improvement over frame-level models", and that the fine-tuning (adaptation) step "improves performance by 15–25% on video retrieval tasks". It gives no frame count per shot, no embedding dimension, and no throughput numbers.
- **The pipeline splits by hardware.** Shot segmentation runs "in parallel on multi-core CPU machines", and the shots are stored as mp4 files in S3. Embedding runs on GPUs, with a data loader that prefetches and preprocesses so the GPU stays busy.
- **CLIP4Clip backs up the pooling choice at small scale.** On small datasets "it is better not to introduce new parameters and adopt a mean pooling mechanism on video frames". Sequential (temporal) models won only on larger datasets (LSMDC, MSR-VTT 9K).
- **Match Cutting shows where pooling falls short.** For motion match cuts, Netflix used optical flow (RAFT) and noted that a temporal average of the frames loses "some valuable information". Even then, top results mostly reflected "similar camera movement" rather than matching action, because background pixels outnumber foreground ones. Among frozen encoders, the video model R(2+1)D did best for motion. For frame match cuts, one middle frame per shot was enough.

**Lessons for a homelab:**

- A Shot as the Segment, embedded by mean-pooling a few CLIP/SigLIP frame embeddings, is a validated baseline. We cannot fine-tune as Netflix did, so expect to give up some of their 15–25% gain.
- Queries that are really about motion ("car chase", "someone falls") are the known weak spot of pooled image embeddings. Put them in the evaluation set (DESIGN D5) and don't try to fix them up front.
- The `notes.md` idea of "15 s sampling for one-take sequences" has no counterpart in these sources. Netflix used Shots without qualification. Long takes remain an open question for our own data.
- Shot segmentation is CPU work and embedding is GPU work, which matches our hardware: mini PCs can segment while the laptop embeds.

## 2. How jobs are made resumable and idempotent

The Netflix posts say less about this than the leads suggested. What they do say:

- **Cosmos** is a forward-chaining rule engine (Plato). It invokes stateless functions (Stratum) through a priority queue (Timestone), and functions are "scaled based on queue size". In each rule section "you typically first record the change in state of the workflow". In other words, progress is persisted state, not in-memory position. Throughput work is judged on "tasks per day" and may "use 'opportunistic' compute resources". The post does **not** describe retries, idempotency, deduplication or checkpointing.
- **Amber** (in "Scaling Media ML"): each Amber Feature owns its computation, storage and trigger, and features form a dependency mesh with recursive resolution. Amber guarantees "immutability, versioning, auditing" of stored values. The motivating failures are useful in their own right. Practitioners "recomputed identical features" for the same asset. Bespoke triggers re-fired when metadata changed. Shared steps (shot segmentation, dedup) were recomputed per pipeline, and memoizing them also kept "shot indices" consistent across algorithms.
- **Immich (homelab scale) shows the failure mode first-hand.** Jobs live in a Redis queue. A maintainer's advice when queued jobs are lost is to "readd those with the 'missing' buttons" ([#10650](https://github.com/immich-app/immich/discussions/10650)). An open, user-filed (not maintainer-confirmed) report, [#32726](https://github.com/immich-app/immich/issues/32726), describes ~15% of an 18.7k-image library left permanently without CLIP embeddings after a restart lost queued jobs, with "no errors" logged. The cause: the work was queued once at import, and nothing derived it again from the database state.

**Lessons for a homelab:**

- Never let the queue be the source of truth. Store per-(Media item, stage, model version) completion in the database. A job is then "compute what's missing": diff the library against that table and enqueue the gap. Run that diff on a schedule and at startup, so a crash, sleep or lost queue heals itself. This matches Immich's "Missing" button, but run automatically.
- Write outputs immutably and keyed by (Media item, Segment, model version), so a re-run overwrites or skips instead of duplicating. That is the Amber/Marken pattern, scaled down to a table with a unique key.
- Make shot segmentation its own memoized stage that every later stage depends on. Then Segment IDs stay stable across embedding models and re-runs, which is Netflix's "shot indices" point.

## 3. How derived data is versioned when a model changes

- **Marken:** "An annotation is an immutable object so the identity of the annotation always includes a version", and the base algorithm schema carries an `algorithmVersion` field. Under future work, the team admits that data from "algorithms which are revised" needs large cleanups "without affecting the service". So even Netflix had no finished answer for retiring old versions as of 2023.
- **Media Understanding Platform:** precomputed serving let them "transparently update the algo data without users knowing about it". Initial backfills had to be throttled to "avoid overwhelming the system with huge amounts of writes".
- **Cosmos:** rules "automatically manage updating existing videos" when a better algorithm lands.
- **Michelangelo:** "a model is identified by its UUID and an optional tag (or alias)". Swapping the model behind a tag needs no client change.
- **Immich:** changing the CLIP model means clicking "All" on Smart Search to re-embed everything. The docs warn the switch "might leave bits of the old model's incompatible data in the database". This is the simple approach (wipe and redo, with search degraded meanwhile), and it shows the risk of storing no version.

**Lessons for a homelab:**

- Tag every embedding row with the model ID and version. Build the new version's index next to the old one, and switch an "active model" pointer only when the backfill is complete (Michelangelo's tag, Marken's versioned identity). Search stays up during a multi-day re-embed on one GPU.
- Throttle and prioritize the backfill (newest or most-watched Media items first). Netflix needed this at its scale, and on one laptop GPU a full re-embed could take days.

## 4. How the batch and serving paths stay on the same model

- **Netflix in-video search:** the query is embedded with "the text encoder from the same trained model" that produced the shot embeddings. The two encoders are one artifact.
- **Michelangelo:** "the same expressions are applied at training time and at prediction time", and "the same data and batch pipeline is used for both training and serving". The same model is deployed to an offline Spark container (batch) and to an online serving cluster. One model artifact, two deployments.
- **Immich:** ML can run on a remote, more powerful host, with an advisory that "version mismatches between both hosts may cause bugs". With several ML URLs, they are "tried sequentially", falling back to the local container. That is a direct template for "GPU laptop when awake, mini PC otherwise". The docs do not say whether text queries are embedded by the same remote service, so check that before copying the pattern.

**Lessons for a homelab:**

- Ship one model artifact with a pinned ID, used by both the indexer (GPU) and the query encoder (which can run on CPU). The text tower of CLIP/SigLIP is small enough for CPU query encoding, so the latency path need not wake the laptop. *(This last claim is my inference, not from the sources. Measure it in ADR-002's experiment.)*
- At query time, refuse to search an index whose model ID differs from the query encoder's. This turns silent garbage into a visible error.

## Open questions surfaced

1. **Long takes / one-shot sequences:** no source covers a Segment cap for very long Shots. It needs our own measurement.
2. **Motion-heavy queries:** pooled image embeddings handle these poorly (Match Cutting). Do we accept that, or add a video encoder later?
3. **CPU query encoding latency** for the candidate text encoder on a mini PC: this is the key measurement for ADR-002.
4. **Shot detector choice and cost on CPU:** Netflix names PySceneDetect and TransNet v2 as open-source options. Which to use, and at what throughput on a mini PC?
5. **Retiring old model versions:** how long to keep the previous index after a pointer switch (disk on the homelab).
