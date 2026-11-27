# How other multimodal video search systems work, and how they are evaluated

Resolves issue #13. Feeds [issue #6, Evaluation plan and success metrics](https://github.com/ArjunVenat/Jellyfin-Smart-Search/issues/6) (DESIGN.md D4/D5) and the open Segment decision. Researched 2026-11-26.

This doc builds on [prior-art.md](https://github.com/ArjunVenat/Jellyfin-Smart-Search/blob/research/prior-art/docs/research/prior-art.md) (issue #4) and doesn't repeat it. That doc already covers Netflix in-video search (per-Shot CLIP-style embeddings, fine-tuned), Match Cutting, CLIP4Clip mean pooling, and Immich's job and versioning behaviour.

Terms follow `GLOSSARY.md`. A **Moment** is a result: a Media item plus one timestamp. A **Segment** is the indexed unit. **Shot** and **Scene** keep their film meanings. Benchmarks label *spans* (start and end), which is not the same as our point-like Moment. Section 4 deals with that gap.

Confidence markers:
- **[verified]** means read in the primary source.
- **[vendor claim]** means marketing copy with no technical detail.
- **[unverified]** means I couldn't confirm it.

---

## 1. General systems for searching inside video in plain words

### 1.1 Comparison

| System | What is indexed (the Segment) | Signals / models | How signals are fused | What a result is |
|---|---|---|---|---|
| **Twelve Labs Marengo 3.0** (also on AWS Bedrock) | Clips. The default for video is **dynamic segmentation by shot-boundary detection**, with `minDurationSec` 1–5 (default 4). Optional `fixed` segmentation uses `durationSec` 1–10 (default 6). An optional whole-asset embedding is also available. Up to 4 h of video per file. [verified] | Three embedding options per clip: `visual`, `audio` (non-speech), and `transcription` (speech). 512-d in 3.0, down from 1024-d in 2.7. 2.7 embeddings are **not compatible** with 3.0. [verified] | Either `separate_embedding` (one vector per modality) or `fused_embedding` ("a weighted fusion of multiple embedding modalities"; weights not documented). In the search API, `search_options` (`visual`, `audio`, `transcription`) are combined with `operator` `or` (default) or `and`. [verified] | A clip: `video_id`, `start`, `end` (seconds), and `rank` ("lower numbers indicate higher relevance"). No score is exposed. Results can be grouped by video. Max 50 per page. [verified] |
| **Google Vertex AI `multimodalembedding@001`** | Fixed windows. `intervalSec` ≥ 4, default 16 s. Only 2 min is analysed per request, so longer videos need multiple calls. [verified] | 1408-d by default (128/256/512 optional). **Ignores audio.** [verified] Image and text share one space. The docs imply, but don't state, that video shares it too. [unverified] The newer `gemini-embedding-2` handles audio only when `audio_track_extraction` is on. [verified] | None. Visual only. | One embedding per window. Search is up to you. |
| **Google Video Intelligence API** | Shot changes, plus frame-level analysis by default. [verified, shot-change page only] | Classic per-feature annotators. | None. It has no natural-language search; it produces annotations you search yourself. | Annotations with time offsets. |
| **Amazon Nova Multimodal Embeddings** (Bedrock) | Fixed windows. `segmentationConfig.durationSeconds` 1–30, default 5. Async jobs handle up to 2 h of video. [verified] | 3072-d by default (256/384/1024 optional). [verified] | `AUDIO_VIDEO_COMBINED` gives one vector per segment for sound and picture together. `AUDIO_VIDEO_SEPARATE` gives two. Index-time and query-time `embeddingPurpose` differ (`GENERIC_INDEX` vs. `VIDEO_RETRIEVAL` and others), so the encoder is asymmetric. [verified] | `segmentStartSeconds`/`segmentEndSeconds` per embedding. [verified] |
| **Azure AI Video Indexer** | Shots, Scenes ("a series of consecutive shots, which are semantically related"), and keyframes. [verified] | A bank of separate models: transcript, OCR, labels, faces, observed people and clothing, tracked objects, audio effects, editorial shot type, rolling credits, and more. [verified] | Search runs over the **extracted insights (text and metadata)**, e.g. "finding moments in a video where a person spoke certain words or when two people were seen together". It is not an embedding search. [verified] | Insight occurrences with timestamps. |
| **Immich** (self-hosted photo/video library) | **One embedding per asset.** For a video, the CLIP job encodes the asset's single *Preview* image file. [verified in source: `smart-info.service.ts` `handleEncodeClip` → `encodeImage(previewFile.path)`; `asset-job.repository.ts` `getForClipEncoding` selects `AssetFileType.Preview`] | CLIP family (swappable). OCR is also searchable. [verified] | None. Image-only. | The whole asset. No timestamp inside a video. Per-scene search is an open feature request ([#5936](https://github.com/immich-app/immich/discussions/5936)). |
| **Google Search "key moments"** (YouTube and web video) | Labelled spans, supplied by the publisher (`Clip` markup with `startOffset`/`endOffset`, or YouTube description timestamps) or detected automatically (`SeekToAction`). Publisher marks take priority. [verified] | Not disclosed. | n/a | A label plus a start timestamp. |
| **Lighthouse** (open-source moment-retrieval library, LINE) | 2-s clips within one video of up to 150 s, a limit inherited from the benchmarks. [verified] | Moment-DETR, QD-DETR, EaTR, CG-DETR, UVCOM, TR-DETR, TaskWeave, R2-Tuning. Features: CLIP, CLIP+SlowFast, CLIP+SlowFast+PANNs (audio). [verified] | Learned cross-modal transformer. | `pred_relevant_windows: [[start, end, score], ...]` plus per-clip saliency. [verified] |
| **TVR's XML model** (research reference for VCMR with subtitles) | 1.5-s clips. [unverified exact clip length] | Video features plus subtitle text. | **Late fusion.** The query is split by learned attention into a "video query" and a "subtitle query". Each is scored against its own modality, and the clip score is the average of the two. Inference is two-stage: the top 100 videos first, then a span inside them. [verified] | A span in a video from the corpus. |

Netflix in-video search (per-Shot embeddings, the same model's text encoder for queries) is in prior-art.md §1. I didn't find primary docs on a YouTube in-video natural-language search feature. [not covered]

### 1.2 Patterns across the general systems

1. **Shot-aligned Segments are the commercial default.** Twelve Labs defaults to shot-boundary segmentation with a ~4 s minimum. Netflix uses Shots. Azure exposes both Shot and Scene. Google and Amazon embedders use fixed windows (4–16 s and 1–30 s) and leave the Segment choice to you.
2. **Speech is a separate modality, not an afterthought.** Twelve Labs has a separate `transcription` vector. TVR/XML uses a separate subtitle stream. Azure works mainly off transcript and OCR. Vertex `multimodalembedding@001`, the CLIP-style option closest to what we'd self-host, **ignores audio completely**.
3. **The main fusion pattern is late fusion of per-modality scores.** XML averages a video score and a subtitle score. Twelve Labs offers per-modality search combined with `or`/`and`, and alternatively a learned weighted fused vector. Nobody in these sources does early fusion of raw features for corpus-scale search.
4. **A result is a span (start, end) inside one Media item, ranked.** Twelve Labs returns `start/end/rank`, Lighthouse `[start, end, score]`, Nova `segmentStart/EndSeconds`. Only Immich returns a whole asset, which is the failure mode we exist to fix.
5. **Embeddings are model-locked.** Twelve Labs 2.7→3.0 and Frigate jinav1→v2 both require full re-embedding. This confirms prior-art.md §3: store a model version on every vector.

---

## 2. CCTV / surveillance natural-language search

### 2.1 Self-hosted: Frigate NVR

- **Unit = tracked object, not a frame or Shot.** Frigate's detector tracks an object (person, car, dog, ...) through its lifecycle. Semantic search embeds **that object's thumbnail**: it "encode[s] the thumbnail image and store[s] it in the database". [verified, [semantic search docs](https://docs.frigate.video/configuration/semantic_search)]
- **Models:** Jina CLIP v1 (`small` is quantized and CPU-OK, `large` uses the GPU) or Jina CLIP v2 (multilingual, 89 languages, needs much more RAM/GPU; "CPU inference (with the `small` model) using V2 is not recommended"). Minimum 8 GB RAM and AVX/AVX2. Switching model requires a reindex. [verified]
- **Two searchable fields per object.** A text query is matched against (a) the **thumbnail** embedding (text→image) and (b) an embedding of the **description** (text→text). The Explore UI lets the user pick "Search Type: thumbnail or description". "Find Similar" does image→image. [verified, [Explore docs](https://docs.frigate.video/usage/explore)]
- **Descriptions come from a vision-LLM.** They are generated "at the end of the tracked object's lifecycle". By default Frigate sends **multiple frames from the object's lifetime**, so the model can judge movement and direction. `use_snapshot` sends one higher-quality frame instead, but "limits the model's ability to judge movement or direction". Prompts can be set per object type (`person`, `car`) with `{label}`, `{sub_label}`, `{camera}` placeholders. Providers are pluggable (Ollama/llama.cpp, cloud). Running the vision-LLM on CPU is "not recommended". [verified, [object descriptions](https://docs.frigate.video/configuration/genai/genai_objects)]
- **Review summaries:** a second, event-level description. When a *review item* (an alert or detection episode) ends, Frigate sends up to **20 frames** (preview frames, or recordings at 480 px height) and stores structured JSON: `title`, `scene` (narrative), `shortSummary`, `confidence`, `potential_threat_level`, `other_concerns`. A time-range endpoint can summarise many items into one report. [verified, [review summaries](https://docs.frigate.video/configuration/genai/genai_review)] The docs don't say whether review summaries are themselves searchable. [unverified]
- **Structured filters sit beside the semantic score:** camera, label, sub-label (e.g. a recognised face), attributes, licence plate, zone, date range, time-of-day, min/max score, min/max speed, has-clip/snapshot. [verified]
- **A result is a tracked object.** Its detail pane has snapshot, lifecycle and metadata. "Clicking a moment plays that part of the recording with the bounding box overlaid." [verified]

### 2.2 Commercial video-management systems

| Vendor | What's documented | Confidence |
|---|---|---|
| **Verkada** (AI-Powered Search, launched 2024) | Cameras detect people and vehicles and produce image crops ("hyperzooms"). An **open-source CLIP** model (variant not named) embeds them, served on NVIDIA Triton. The query is embedded with CLIP's text encoder. Search uses **exact k-NN** (not approximate) with **dynamic score thresholding**. Data is sharded by org → date → time. Search pods load only the requested time range, then **filter by camera ID and object type before the matrix multiply**. Scale: >800 M embeddings/day. Guardrails: query moderation and admin-visible audit logs. The result format isn't specified ("relevant clips"). | [verified, [engineering post](https://verkada.com/blog/scaling-natural-language-video-search-inside-verkadas-breakthrough)] |
| **Avigilon** (Motorola Solutions) | The documented natural-language feature is **on-premise genAI "Visual Alerts"**: "use simple, natural language to create alerts". *Appearance Search* finds "a person or vehicle in recorded video" from an example. It's Enterprise-only. The product page doesn't show free-text natural-language *search* over recordings. | [verified for alerts; NL search unverified] |
| **BriefCam** (now part of Milestone) | Search is **filter-based**: object class, appearance, direction, and custom-trained classes, plus appearance similarity and **video synopsis** (overlays many events from different times into one short clip). The product page doesn't mention natural-language search. | [verified, [Milestone BriefCam page](https://www.milestonesys.com/products/software/briefcam/)] |
| **Rhombus** (AI Search, beta from 2025-04-15) | Plain-English queries across chosen cameras. Results are "footage snippets", filterable by time and camera. Coverage mentions a ChatGPT integration. Neither models nor the index unit are disclosed. | [vendor claim, [blog](https://www.rhombus.com/blog/introducing-ai-search)] |
| **Spot AI** | Edge appliances "read" footage to make it word-searchable (TechCrunch, 2023). No technical documentation found. | [vendor claim / unverified] |

### 2.3 How CCTV search differs from film/TV search

| | CCTV / NVR | Film / TV (this project) |
|---|---|---|
| Camera | Fixed, so a background model and motion masks work. | Moving, with cuts. Every Shot has a new background. |
| Natural unit | **Tracked object** (detector + tracker), grouped into events/review items. | **Shot / Scene**. Cuts are the boundaries; there is no object identity across cuts. |
| Typical query | Attribute-of-one-object ("person in red jacket", "white van"). | Whole-frame semantics, action, setting, mood, dialogue. |
| What gets embedded | A **crop** of the object (Verkada, Frigate thumbnail). This avoids the background-dominates problem prior-art.md noted for Match Cutting. | The whole frame or Shot. |
| Audio / speech | Mostly absent or ignored. | Subtitles are often the strongest signal (see TVR, §3.1). |
| Strong metadata filters | Camera, zone, time-of-day, speed, plate. | Series/season/episode, year, genre, people (Jellyfin metadata). |
| Result | Object track with lifecycle, opened in the recording at that time with a bounding box. | Moment = Media item + timestamp, opened in the player. |

### 2.4 What this project could borrow from CCTV systems

1. **Store two fields per Segment, image embedding and text description, and search both.** Frigate's thumbnail-vs-description split maps onto Segment-frame-embedding vs. Segment caption (a VLM caption or the subtitle text).
2. **Caption from several frames, not one.** Frigate's own docs say a single snapshot "limits the model's ability to judge movement". That fits prior-art.md's note that motion queries are the weak spot.
3. **Event-level summaries as a coarser layer.** Frigate's review summaries (≤20 frames → `title` + `scene` narrative) are the CCTV analogue of a **Scene** summary. They are a cheap second index level above Shot Segments.
4. **Filter before scoring, then exact k-NN.** Verkada filters by time range, camera and object type *before* exact search. At homelab scale (a few hundred thousand Segment vectors, my rough estimate), exact search after a metadata pre-filter (Media item, series, season) is feasible and avoids ANN recall loss. *(The scale figure is my inference, not from the sources.)*
5. **Dynamic score thresholding** (Verkada) rather than a fixed top-k suits many-answer queries, where the number of true hits varies per query.
6. **Optional, later: object crops for attribute queries.** Running a detector on keyframes and embedding person/vehicle crops would help "man in a red coat" queries, where the whole-frame embedding is dominated by background. It is not essential for v1.

---

## 3. How moment retrieval is evaluated

### 3.1 The benchmarks

| Benchmark | Task | Data | How queries and labels were made | Answers per query | Standard metrics |
|---|---|---|---|---|---|
| **TVR** (Lei et al., ECCV 2020) [arXiv 2001.09099](https://arxiv.org/abs/2001.09099) | **VCMR** (find the moment across the whole corpus) plus single-video retrieval. **Uses subtitles.** | 109K queries, 21.8K videos from 6 TV shows. Mean video 76.2 s, mean moment 9.1 s. | AMT workers who passed a 12-question qualification (9 of 12 to pass; 67% of 1,055 passed). Each worker writes a query, tags it *video-only / sub-only / video+sub*, and marks start and end. Queries need ≥8 words, can't be copied from subtitles, and **must uniquely match one moment in the video**. Post-check on 3,600 pairs: 92% rated at least "neutral". ~91% of queries need video and ~26% need subtitles. | **One** per query (by design). | R@{1,5,10,100} at temporal IoU {0.5, 0.7}. A hit needs the correct video **and** IoU ≥ threshold. |
| **TVR-Ranking** (Liang et al., 2024) [arXiv 2407.06597](https://arxiv.org/abs/2407.06597) | Ranked VMR over a corpus with **imprecise** queries. | TVR moments. 94,442 query–moment pairs judged by humans. | Character names in TVR queries were replaced with pronouns (an LLM plus NER/similarity checks), making queries vague the way real users' are. Each candidate moment (≤40 per query, pre-ranked by caption similarity) got a 0–4 relevance score from 2 annotators, with 2 more if they disagreed. | **Many**: ~27 relevant moments per query on average. | **NDCG@{10,20,40} at IoU ≥ {0.3,0.5,0.7}.** A prediction takes the grade of its best-IoU unmatched ground truth if IoU ≥ μ, otherwise 0. Matched ground truths are removed (no double credit). |
| **QVHighlights** (Lei et al., NeurIPS 2021) [arXiv 2107.09609](https://arxiv.org/abs/2107.09609) | Single-video moment retrieval plus highlight detection. | 10,310 queries, 10,148 videos (150-s cuts of 5–30 min YouTube videos), 18,367 moments. | Workers write a free-form query, then tick **2-s clips** in a grid; all relevant clips are marked. Each relevant clip is rated on a 5-point saliency scale by 3 workers. On 600 pairs, ~90% of queries had mean inter-annotator IoU > 0.9. | **Many**: avg 1.8 disjoint moments per query. | R1@{0.5,0.7}; mAP@0.5, mAP@0.75, and **average mAP over IoU 0.5:0.05:0.95**; highlight mAP and HIT@1 (top clip rated "Very Good"). |
| **Charades-STA** (Gao et al., ICCV 2017) [arXiv 1705.02101](https://arxiv.org/abs/1705.02101) | Single-video grounding. | 13,898 train / 4,233 test pairs on indoor, scripted, ~30-s Charades videos. | **Semi-automatic**: Charades paragraph descriptions are split at conjunctions, matched by keyword to activity labels, and given those labels' times. Two co-authors checked them. | One. | "R@n, IoU=m": the share of queries with ≥1 of the top-n above IoU m. n ∈ {1,5}, m ∈ {0.5,0.7}. |
| **ActivityNet Captions** (Krishna et al., ICCV 2017) [arXiv 1705.00754](https://arxiv.org/abs/1705.00754) | Built for dense captioning; reused for grounding. | 20K videos (~180 s mean), ~100K sentences, ~3.65 per video. | AMT workers write a paragraph (one sentence per event), then mark each sentence's start and end. ~10% of events overlap. | One per sentence. | Grounding papers use R@n at IoU {0.3,0.5,0.7}. The original dense-captioning task uses AP at tIoU {0.3,0.5,0.7}. |
| **DiDeMo** (Hendricks et al., ICCV 2017) [arXiv 1708.01641](https://arxiv.org/abs/1708.01641) | Single-video grounding. | 10,464 videos (25–30 s), 40,543 descriptions. | Video is split into **fixed 5-s segments**. One annotator describes a moment so others could find it. 3 more annotators localise it, and it is kept if ≥3 of 4 agree within one segment. | One, but **4 human labels** are kept. | Rank@1, Rank@5, mIoU. Scored against each annotator, using the best 3 of 4, which tolerates label ambiguity. |
| **MAD** (Soldan et al., CVPR 2022) [arXiv 2112.00431](https://arxiv.org/abs/2112.00431) | Grounding in **full-length movies**. | 384K sentences, ~1,200 h, 650 movies (mean ~111 min). Mean moment **4.1 s**. | Sentences come from professional **audio descriptions** for blind viewers, aligned to the film and transcribed. No crowd query writing. | One. | R@{1,5,10,50,100} at IoU {0.1,0.3,0.5}. Low IoU thresholds because moments are tiny relative to the video. |
| **MSR-VTT** (Xu et al., CVPR 2016) [MSR page](https://www.microsoft.com/en-us/research/publication/msr-vtt-a-large-video-description-dataset-for-bridging-video-and-language/) | **Clip-level** text→video retrieval (no localisation). | 10K clips (10–32 s), ~20 captions each (200K total, 1,327 AMT workers). Videos came from 257 popular search-engine queries. | Workers caption already-trimmed clips. | One clip per caption, though in practice many near-duplicates. | R@{1,5,10}, median rank (MdR), mean rank (MnR) on the "1k-A" test split (1,000 clip–text pairs), as in CLIP4Clip. |
| **TRECVID AVS** (NIST) [2025 overview](https://trec.nist.gov/pubs/trec34/papers/Overview_avs.pdf) | Ad-hoc search: return **Shots** matching a topic from ~1000 h (V3C). | — | NIST assessors judge sampled shots yes/no, watching the whole shot with audio. | **Many**. | Mean extended inferred AP (xinfAP), computed with `sample_eval` on stratified samples of pooled results. |

TRECVID's sister event, the Video Browser Showdown, runs **known-item search**, which is the closest academic analogue of "I remember this moment, find it". I couldn't verify its scoring rules from a primary source. [unverified]

### 3.2 Hit definitions in use

- **Temporal IoU against a labelled span** is the near-universal hit rule. Thresholds are **0.5 and 0.7** for short videos (TVR, Charades-STA, QVHighlights), **0.3/0.5/0.7** for ActivityNet, and **0.1/0.3/0.5** for movie-length MAD, where 4-s moments sit inside 2-h videos.
- **A corpus hit needs the right video too** (TVR VCMR).
- **Segment-grid labels**: DiDeMo (5-s segments) and QVHighlights (2-s clips) label on a fixed grid. That makes "hit = same or overlapping grid cell" equivalent to an IoU rule at grid resolution.
- **Point-with-tolerance** hits (a timestamp within ±N s), which is how our glossary defines a Moment, are **not used by any of these benchmarks**. The nearest relatives are DiDeMo's "within one segment" annotator-agreement rule and the TVR verification finding that "strongly disagree" labels were often off by ≤1 s.
- **Boundary-aware discounting**: Yuan et al. ([arXiv 2101.09028](https://arxiv.org/abs/2101.09028)) propose dR@n,IoU@m, which multiplies each hit by a factor that falls with start and end distance from the truth. They show that R@n at **small** IoU is "unreliable" on biased datasets.

### 3.3 Metrics by query type

| Query type | Metrics used | Notes |
|---|---|---|
| Single answer (TVR, Charades-STA, MAD, DiDeMo) | R@k at IoU, MdR/MnR (MSR-VTT). | MRR is the natural rank summary for single-answer queries, but these benchmarks report R@k rather than MRR. [no benchmark above reports MRR] |
| Many answers, binary relevance (QVHighlights, TRECVID AVS) | mAP at IoU (averaged over thresholds), xinfAP. | |
| Many answers, graded relevance (TVR-Ranking) | NDCG@k at IoU ≥ μ, with one-to-one matching so a system can't score twice on one truth. | This is the only benchmark here built for vague, real-user-like queries. |

### 3.4 Known biases and pitfalls

- **Temporal position priors.** Charades-STA moments mostly start near the beginning and last ~20% of the video. In ActivityNet, 40% of queries cover >30% of the video, so a "predict everything" baseline scores well at low IoU (Yuan et al.). TVR's own paper notes more moments near the start and says a frequency baseline beats chance for that reason. QVHighlights and MAD were designed to have near-uniform positions.
- **Blind baselines are competitive.** Otani et al. ([arXiv 2009.00325](https://arxiv.org/abs/2009.00325)) show query-only baselines (a moment prior plus the first verb) beating several published deep models on Charades-STA. When video frames are **shuffled**, 2D-TAN's predictions barely change (67.9% identical on ActivityNet). So the models leaned on language and position, not vision.
- **Verb concentration.** The top 30 verbs cover 93.2% of Charades-STA queries (51.4% for ActivityNet), so queries are easy to predict from wording.
- **Single-reference labels undercount correct answers.** With 5 re-annotators, human R@1,IoU>0.5 was only 52.1 (Charades) and 44.4 (ActivityNet) against the original label. That is about the same as the models, because several moments fit one query (Otani et al.). DiDeMo keeps multiple labels for this reason, and TVR-Ranking reports ~27 relevant moments per vague query.
- **"Unique moment" query writing inflates specificity.** TVR required each query to match exactly one moment, which pushed workers toward long, name-heavy descriptions. TVR-Ranking had to strip character names to make queries realistic.
- **Writers watched the video first.** In every crowd benchmark above, the writer sees the moment and then describes it. That gives detailed, visually grounded queries, unlike recall-based queries from memory.
- **Semi-automatic labels** (Charades-STA's keyword matching) carry activity-label noise.
- **Absolute VCMR numbers are tiny.** XML's TVR VCMR R@1 at IoU 0.7 is 3.25 (test-public). Corpus-scale span retrieval at strict IoU is very hard, so don't read low early numbers as failure.

---

## 4. Implications for this project

### For the evaluation plan (issue #6, DESIGN D4/D5)

1. **Label spans, score points.** Our Moment is a timestamp, but benchmark practice and ambiguity argue for **labelling each relevant Moment as a span** (start, end) and accepting multiple spans per query (QVHighlights, TVR-Ranking). Then define **hit = returned timestamp falls inside a labelled span, widened by ±τ seconds**. τ ≈ 2–5 s is a reasonable starting range given 2-s and 5-s label grids and the ~1-s labelling error seen in TVR. τ is a free choice; pick it and freeze it. Don't use IoU as the primary rule: a point has no IoU. Report IoU only if a Segment span is shown to the user. *(The point-plus-tolerance rule is my proposal; no benchmark here uses it.)*
2. **Use the right Media item as a separate, lenient metric.** Report "right Media item in top-k" alongside "right Moment in top-k", as TVR splits video retrieval from VCMR. This shows whether failures are in finding the item or finding the moment within it.
3. **Metrics by query type:**
   - Single-answer ("the scene where X says Y"): **Recall@{1,5,10}** and **MRR**.
   - Many-answer ("explosions", "beach at sunset"): **nDCG@10** with one-to-one matching (TVR-Ranking style), or AP if labels are binary.
   - Keep the two groups in separate rows. Don't average them together.
4. **Write some queries from memory, and some after viewing.** Benchmarks only use view-then-describe queries, which are more specific than real use. Tag each query with how it was written, plus a TVR-style modality tag (visual / dialogue / both) and a type (object, action/motion, setting, mood, quote). Report per tag, because TVR showed single-modality models fail on the other modality's queries.
5. **Run a blind baseline and a shuffled-frames check.** Following Otani et al., include (a) a **position prior / random-timestamp baseline** in the right Media item and (b) a **frames-shuffled** run of the visual pipeline. If the visual system barely beats (a) or barely changes under (b), it isn't using the picture.
6. **Mind position bias in our own labels.** Cold opens, credits and recaps sit at fixed positions in episodes. Check that labelled Moments aren't clustered there, and exclude credits (Azure even has a rolling-credits detector for this).

### Baselines every option should be compared against

| Baseline | Why |
|---|---|
| **Subtitle-only BM25** on subtitle cues, with Moment = cue start | The TVR result: most dialogue queries are solved by text. It's cheap and needs no GPU. |
| **Subtitle-only dense embedding** (same text encoder as the query) | It separates the lexical-match effect from the semantic effect. |
| **Visual-only CLIP/SigLIP, one keyframe per Segment** | The Netflix and Immich floor. |
| **Visual-only, mean-pooled frames per Shot** | prior-art.md's recommended baseline (CLIP4Clip). |
| **Late fusion** of visual and subtitle scores (simple average, or `max` vs. `and`/`or`) | This is what XML and Twelve Labs do. It is the obvious first multimodal system to beat. |
| **Random-timestamp in the correct Media item** and a **position prior** | The bias floors from §3.4. |

### For the Segment and index design

- **Shot-aligned Segments with a minimum length** (Twelve Labs: ~4 s minimum) are the commercial default. A fixed 4–16 s window (Vertex, Nova) is the simple alternative. Both are worth comparing on the eval set.
- **Keep subtitles as their own modality and their own scores.** Then fuse late. Don't count on a CLIP-style video embedder to pick up speech; Vertex's ignores audio altogether.
- **Two text-side fields per Segment** (subtitle text and a multi-frame VLM caption) plus the image embedding, as in Frigate. Store the model version on all three.
- **Pre-filter by Jellyfin metadata, then run exact k-NN** (Verkada). Use a score threshold rather than a fixed k for many-answer queries.

### Open questions surfaced

1. Tolerance τ for a point-in-span hit: check against our own labelling consistency, e.g. label a few queries twice and measure disagreement.
2. Should vague, many-answer queries be graded 0–4 (nDCG) or labelled binary (AP)? Graded labels cost more to make.
3. Whether a Scene-level summary layer (Frigate review-summary style) pays off over Shot-level captions alone.
4. Video Browser Showdown known-item-search scoring is unverified, and it may be the best model for "find the moment I remember".

## Sources

General systems:
[Twelve Labs Marengo overview](https://docs.twelvelabs.io/docs/concepts/models/marengo) ·
[Twelve Labs search API](https://docs.twelvelabs.io/v1.3/api-reference/any-to-video-search/make-search-request) ·
[Marengo 3.0 on Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-marengo-3.html) ·
[Vertex multimodal embeddings](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-multimodal-embeddings) ·
[Video Intelligence shot change](https://docs.cloud.google.com/video-intelligence/docs/feature-shot-change) ·
[Nova embeddings schema](https://docs.aws.amazon.com/nova/latest/userguide/embeddings-schema.html) ·
[Azure AI Video Indexer overview](https://learn.microsoft.com/en-us/azure/azure-video-indexer/video-indexer-overview) ·
[Immich searching docs](https://docs.immich.app/features/searching/) ·
Immich source (`server/src/services/smart-info.service.ts`, `server/src/repositories/asset-job.repository.ts`, main @ 532707a) ·
[Google key moments](https://developers.google.com/search/docs/appearance/structured-data/video) ·
[Lighthouse](https://github.com/line/lighthouse)

CCTV:
[Frigate semantic search](https://docs.frigate.video/configuration/semantic_search) ·
[Frigate object descriptions](https://docs.frigate.video/configuration/genai/genai_objects) ·
[Frigate review summaries](https://docs.frigate.video/configuration/genai/genai_review) ·
[Frigate Explore](https://docs.frigate.video/usage/explore) ·
[Verkada engineering post](https://verkada.com/blog/scaling-natural-language-video-search-inside-verkadas-breakthrough) ·
[Avigilon cloud services](https://www.avigilon.com/products/acc/avigilon-cloud-services) ·
[BriefCam (Milestone)](https://www.milestonesys.com/products/software/briefcam/) ·
[Rhombus AI Search](https://www.rhombus.com/blog/introducing-ai-search) ·
[Spot AI (TechCrunch)](https://techcrunch.com/?p=2218606)

Evaluation:
[TVR](https://arxiv.org/abs/2001.09099) ·
[TVR-Ranking](https://arxiv.org/abs/2407.06597) ·
[QVHighlights / Moment-DETR](https://arxiv.org/abs/2107.09609) ·
[Charades-STA / TALL](https://arxiv.org/abs/1705.02101) ·
[ActivityNet Captions](https://arxiv.org/abs/1705.00754) ·
[DiDeMo](https://arxiv.org/abs/1708.01641) ·
[MAD](https://arxiv.org/abs/2112.00431) ·
[MSR-VTT](https://www.microsoft.com/en-us/research/publication/msr-vtt-a-large-video-description-dataset-for-bridging-video-and-language/) ·
[CLIP4Clip (MSR-VTT protocol)](https://arxiv.org/abs/2104.08860) ·
[Otani et al., hidden challenges](https://arxiv.org/abs/2009.00325) ·
[Yuan et al., closer look](https://arxiv.org/abs/2101.09028) ·
[TRECVID 2025 AVS overview](https://trec.nist.gov/pubs/trec34/papers/Overview_avs.pdf)
