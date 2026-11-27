# Research: embedding models and captioners that fit 8 GB VRAM

*Resolves issue #5. Feeds the "Embedding model choice" ticket (#9), which makes the actual pick. Researched 2026-11-26.*

**Question.** Which models are realistic v1 candidates on one RTX 4060 Laptop GPU (8 GB VRAM), with query-time text encoding possibly on a CPU-only mini PC? Covers image–text embedders, video/multi-frame embedders, captioners/small VLMs, and text embedders for subtitles.

**How to read the numbers.**
- **[src]** means the number is quoted from the linked primary source.
- **[est]** means I estimated it. The method is given in [Estimation method](#estimation-method). Treat every [est] as a hypothesis for the #9 prototype to measure.
- Almost no model card gives VRAM or throughput on a laptop GPU. Most VRAM and speed figures below are therefore [est].

---

## TL;DR shortlist

| Role | Candidate | Why it's on the list |
|---|---|---|
| Image–text embedder (fast) | **SigLIP 2 B/16** (256 or 384 px) | Apache-2.0, 768-d. Flickr T→I R@1 81.7–83.8 [src]. About 100–400 frames/s on the 4060 [est]. Cheap text tower on CPU. |
| Image–text embedder (quality) | **SigLIP 2 So400m** (/14 or /16, 384 px) | Best retrieval scores that fit easily: COCO T→I R@1 55.8–56.0 [src]. 1152-d. About 20–26 frames/s [est]. Heavy text tower on CPU (~0.25 s+ per query [est]). |
| Image + video embedder | **Perception Encoder PE-Core L/14-336** | Apache-2.0, 1024-d. Video by averaging 8 frame embeddings, with MSR-VTT T2V 50.3 published [src]. This gives some motion signal with no extra model. |
| One model for everything | **EmbeddingGemma 2** (740M) | Apache-2.0, 768-d. Text, image, video (1 fps) and audio in one space [src]. New (Oct 2026). No COCO/Flickr numbers to compare against CLIP-family models, so it must be measured. |
| Captioner (affordable at scale) | **Florence-2 base / large** | MIT, 0.23B / 0.77B [src]. The only captioner here small enough to caption every Segment in a large library [est]. |
| Captioner (rich, selective use) | **Qwen3-VL-2B**, **SmolVLM2-2.2B**, **Moondream2** | All Apache-2.0 and about 2B params. Fit in 8 GB at bf16. Too slow to caption every Segment [est], so best kept for re-ranking or a subset. |
| Subtitle text embedder | **Qwen3-Embedding-0.6B** or **EmbeddingGemma-300M**; **all-MiniLM-L6-v2** as a CPU baseline | Strong MTEB scores [src]. MRL lets you truncate dims. Small enough for CPU. |

Not shortlisted: OpenCLIP ViT-B/32 is useful only as a baseline. MobileCLIP2 is fast, but its weights use Apple's restrictive research terms. DFN ViT-H/14 is beaten by So400m at lower cost. VideoPrism is JAX-only. Qwen2.5-VL-3B is about 4B params and uses a non-commercial license. Details below.

---

## 1. Image–text embedders

### 1.1 Retrieval quality (zero-shot, recall@1 %)

| Model | Res | Embed dim | COCO T→I | COCO I→T | Flickr T→I | Flickr I→T | Source |
|---|---|---|---|---|---|---|---|
| OpenCLIP ViT-B/32 laion2b_s34b_b79k | 224 | 512 | 39.3 | 56.3 | 66.8 | 84.1 | [OpenCLIP results][oc-res] |
| OpenCLIP ViT-B/16 dfn2b | 224 | 512 | 43.4 | 60.4 | 69.1 | 85.4 | [OpenCLIP results][oc-res] |
| OpenCLIP ViT-L/14 datacomp_xl | 224 | 768 | 45.7 | 63.3 | 73.4 | 89.0 | [OpenCLIP results][oc-res] |
| EVA02-L/14-336 merged2b | 336 | 768 | 47.9 | 64.2 | 78.0 | 89.6 | [OpenCLIP results][oc-res] |
| DFN ViT-H/14-378 | 378 | 1024 | 55.6 | 71.9 | 82.0 | 94.0 | [OpenCLIP results][oc-res] |
| SigLIP So400m/14 (v1) | 384 | 1152 | 54.2 | 72.4 | 83.0 | 94.3 | [OpenCLIP results][oc-res] |
| **SigLIP 2 B/16** | 224 | 768 | 52.1 | 68.9 | 80.7 | 93.0 | [SigLIP 2 paper, Table 1][siglip2] |
| **SigLIP 2 B/16** | 256 | 768 | 53.2 | 69.7 | 81.7 | 94.4 | [SigLIP 2 paper][siglip2] |
| **SigLIP 2 B/16** | 384 | 768 | 54.6 | 71.4 | 83.8 | 94.9 | [SigLIP 2 paper][siglip2] |
| SigLIP 2 L/16 | 256 | 1024 | 54.7 | 71.5 | 84.1 | 94.5 | [SigLIP 2 paper][siglip2] |
| SigLIP 2 L/16 | 384 | 1024 | 55.3 | 71.4 | 85.0 | 95.2 | [SigLIP 2 paper][siglip2] |
| **SigLIP 2 So400m/14** | 384 | 1152 | 55.8 | 71.7 | 85.7 | 94.9 | [SigLIP 2 paper][siglip2] |
| **SigLIP 2 So400m/16** | 384 | 1152 | 56.0 | 71.2 | 85.3 | 95.9 | [SigLIP 2 paper][siglip2] |
| SigLIP 2 g/16 (1B vision) | 384 | 1536 | 56.1 | 72.8 | 86.0 | 95.4 | [SigLIP 2 paper][siglip2] |
| **PE-Core B/16** | 224 | 1024 | 50.9 | — | — | — | [perception_models README][pe-gh] |
| **PE-Core L/14** | 336 | 1024 | 57.1 | — | — | — | [perception_models README][pe-gh] |
| PE-Core G/14 | 448 | 1280 | 58.1 | — | — | — | [perception_models README][pe-gh] |

Notes:
- Embed dims for SigLIP 2 L and g, and for the OpenCLIP models, come from the OpenCLIP [model profile][oc-prof] and model configs. PE dims come from the [PE paper, Table 4][pe-paper].
- COCO and Flickr show web-photo captions. Film frames (dark scenes, odd framing, letterboxing) are a different domain. Treat these scores as ranking hints only. The project's own eval set (DESIGN.md D5) is what decides.
- MobileCLIP2 reports only ImageNet and a 38-dataset average, not COCO/Flickr retrieval in its README. MobileCLIP2-B scores 79.4 % IN-1k and 65.8 % on the 38-dataset average. MobileCLIP2-S4 scores 81.9 % and 67.5 % [[ml-mobileclip][mobileclip]].

### 1.2 Size, VRAM, throughput, license

Image-tower throughput was measured by timm on a desktop **RTX 4090** (AMP, batch 512–1024) [[timm benchmark CSV][timm-gpu]]. The 4060 Laptop column is scaled from it [est].

| Model (timm name for the image tower) | Image params | Text params | 4090 img/s [src] | 4060 Laptop img/s [est] | fp16 weights, image tower [est] | fp16 weights, both towers [est] | License |
|---|---|---|---|---|---|---|---|
| CLIP B/32 (`vit_base_patch32_clip_224`) | 88M | 63M | 10,128 | 1,250–1,700 | 0.18 GB | 0.3 GB | OpenCLIP code MIT |
| CLIP B/16 (`vit_base_patch16_clip_224`) | 87M | 63M | 2,784 | 350–460 | 0.17 GB | 0.3 GB | — |
| SigLIP B/16-224 | 93M | 110M* | 2,439 | 300–400 | 0.19 GB | 0.4–0.8 GB | Apache-2.0 (SigLIP 2) [[HF card][siglip2-b]] |
| SigLIP B/16-256 | 93M | ~0.3B* | 1,875 | 230–310 | 0.19 GB | ~0.8 GB | Apache-2.0 [[HF card][siglip2-b]] |
| SigLIP B/16-384 | 93M | ~0.3B* | 796 | 100–135 | 0.19 GB | ~0.8 GB | Apache-2.0 |
| CLIP L/14-224 | 304M | 124M | 658 | 80–110 | 0.6 GB | 0.86 GB | — |
| SigLIP L/16-256 | 316M | — | 626 | 78–104 | 0.63 GB | — | Apache-2.0 |
| SigLIP So400m/14-224 | 428M | 450M | 464 | 58–77 | 0.86 GB | 1.8 GB | Apache-2.0 |
| CLIP L/14-336 (≈ PE-Core L/14-336 class) | 304M | 124M | 292 | 36–49 | 0.6 GB | 0.86 GB | — |
| SigLIP L/16-384 | 316M | — | 264 | 33–44 | 0.63 GB | — | Apache-2.0 |
| CLIP H/14-224 | 632M | 354M | 346 | 43–58 | 1.3 GB | 2.0 GB | — |
| **SigLIP So400m/14-384** | 428M | 450M | 154 | **19–26** | 0.86 GB | 1.8–2.3 GB | Apache-2.0 [[HF card][siglip2-so]] |

\* SigLIP 2 uses the Gemma tokenizer (256k vocab). The big embedding table inflates the text-tower *param* count, but not its per-token compute much. HF lists `siglip2-base-patch16-256` at 0.4B params total [[HF][siglip2-b]] and `siglip2-so400m-patch16-384` at 1B [[HF][siglip2-so]]. Params and GFLOPs for the v1 SigLIP and CLIP towers come from the [OpenCLIP model profile][oc-prof].

- **SigLIP 2 vs SigLIP v1 timm rows.** The paper presents SigLIP 2 as a drop-in replacement for SigLIP at the same sizes, so the v1 image-tower throughput rows are a fair proxy [[SigLIP 2 paper][siglip2]].
- **PE-Core L/14-336.** 0.32B vision, 0.31B text, 1024-d CLIP dim [[HF card][pe-hf]], [[paper Table 4][pe-paper]]. Its image tower is the same width and depth class as CLIP L/14-336, so expect a similar 36–49 img/s [est]. License: Apache-2.0 for code and PE weights [[repo][pe-gh]]. Note that PLM, a different model in the same repo, uses the FAIR Research License.
- **VRAM verdict.** Every model in this table fits in 8 GB with a large margin. The image tower alone is under 1 GB at fp16. Activations at batch 64–256 add roughly 1–3 GB [est]. VRAM is **not** the binding constraint for embedders. Throughput and decode are.
- **int8.** No primary source gives int8 retrieval numbers for these towers, and at under 1 GB of weights it isn't needed on the GPU. int8 matters mostly for the CPU query path (see §5).
- **MobileCLIP2.** Image + text params run from 11.4+63.4M (S0) to 321.6+123.6M (S4). Latencies are 1.5+3.3 ms (S0) and 19.6+6.6 ms (S4) on a device the README doesn't name; it cites an iPhone 12 Pro Max elsewhere [[ml-mobileclip][mobileclip]]. The code is MIT, but the weights are under the **Apple ML Research Model Terms of Use**, so they're dropped from the shortlist on license grounds. OpenCLIP supports them natively.

### 1.3 Library-scale arithmetic [est]

These figures are illustrative. The library size is an assumption, since DESIGN.md D6 is still TODO.

- **Assume** 1,000 hours of video and one keyframe per Segment at an average of 5 s. That gives ~720k frames.
- At 300 frames/s (SigLIP 2 B/16-224), embedding takes **~40 min** of GPU time.
- At 22 frames/s (So400m-384), it takes **~9 h**.
- At 3 frames per Segment for multi-frame averaging, multiply by 3.
- Either way, the full library can be embedded in one night to a few nights. That means the embedder choice can be driven by quality, and frame decoding on the NFS/mini-PC side will likely become the bottleneck first. That's a question for the decode/ingest ticket, not this one.

---

## 2. Video / multi-frame embedders (motion)

| Model | How it handles video | Retrieval number | Size | Fits 8 GB? | License | Notes |
|---|---|---|---|---|---|---|
| **PE-Core L/14-336** | Uniformly samples 8 frames, encodes each, then averages the embeddings [[PE paper §2.4–2.5][pe-paper]] | MSR-VTT T2V 50.3 (L), 47.6 (B), 51.2 (G) [[README][pe-gh]] | 0.32B vision | Yes, same cost as 8 image embeds [est] | Apache-2.0 | Best fit: one model serves both single-frame Moments and short motion clips. |
| **EmbeddingGemma 2** | Samples frames through the vision encoder at 1 fps by default, 140 tokens per frame, up to ~58 frames in an 8k context [[model card][eg2-card]] | MMEB v2 (Video) Hit@1 50.67 [[model card][eg2-card]]. The paper's MSR-VTT row is 50.90 (setup unverified) [[paper][eg2-paper]] | 740M total, ~440M for text+image | Yes, ~1.5 GB bf16 weights [est] | Apache-2.0 | Not directly comparable to PE's MSR-VTT metric. Recommends bf16 or fp32, not fp16 [[model card][eg2-card]]. The RTX 4060 (Ada) supports bf16. |
| VideoPrism-LvT-B / L | 8–16 frames at 288 px into a video encoder [[repo][videoprism]] | MSR-VTT-1K t2v 50.1 (both public B and L) [[repo][videoprism]] | 248M / 580M | Yes [est] | Code Apache-2.0. The weights license isn't stated explicitly ("other materials" are CC-BY 4.0) | **JAX/Flax only.** PyTorch support is a TODO. Not shortlisted for v1. |
| InternVideo2-Stage2 1B | — | Not on the model card | ~1B (from the name; unverified) | Probably, at fp16 [est] | apache-2.0 on the card, gated | Not investigated further. Larger than the alternatives, and the card has no numbers [[HF][iv2]]. |

**Takeaway.** Averaging a few frame embeddings (PE-Core's published recipe) gives MSR-VTT numbers on par with dedicated video encoders such as VideoPrism, at no extra model cost. That makes "multi-frame mean per Segment" the cheap first experiment for the motion risk in DESIGN.md D16.

---

## 3. Captioners / small VLMs

| Model | Params | VRAM | Per-image latency on 4060 Laptop | Quality signal | License |
|---|---|---|---|---|---|
| **Florence-2-base** | 0.23B [[HF][florence]] | ~0.5–1 GB fp16 [est] | ~0.05–0.15 s per caption [est] | COCO Caption CIDEr 133.0 zero-shot [[HF][florence]] | MIT |
| **Florence-2-large** | 0.77B [[HF][florence]] | ~1.5–2.5 GB fp16 [est] | ~0.1–0.3 s per caption [est] | CIDEr 135.6 zero-shot. Prompts: `<CAPTION>`, `<DETAILED_CAPTION>`, `<MORE_DETAILED_CAPTION>` [[HF][florence]] | MIT |
| Moondream2 (rev 2025-06-21) | 2B, bf16 [[HF][moondream]] | ~4–5 GB [est] | ~0.5–2 s for a short caption [est] | Short, normal and long caption modes. The card has changelog scores only [[HF][moondream]] | Apache-2.0 |
| SmolVLM2-2.2B-Instruct | 2.2B [[HF][smolvlm2]] | **5.2 GB for video inference** [[HF][smolvlm2]] | ~1–3 s [est] | Video-MME 52.1, MLVU 55.2, MVBench 46.27 [[HF][smolvlm2]] | Apache-2.0 |
| Qwen3-VL-2B-Instruct | 2B, bf16 [[HF][qwen3vl]] | ~5–6 GB bf16 [est] | ~1–3 s [est] | Native video understanding. The card's benchmark charts are images; text gives only longshot/PBench [[HF][qwen3vl]] | Apache-2.0 |
| Qwen2.5-VL-3B-Instruct | "3B" in name, 4B params in safetensors [[HF][qwen25vl]] | ~8 GB+ bf16, **so it needs 4/8-bit quantization** [est] | ~2–4 s [est] | Video-MME 67.6/61.5, MVBench 67.0 [[HF][qwen25vl]] | **Qwen Research License, non-commercial only** [[LICENSE][qwen25vl-lic]] |

**Can every Segment be captioned?** [est], same 720k-Segment assumption as §1.3:
- Florence-2-large at ~0.2 s takes **~40 h** of GPU time. That's feasible as a multi-night background job.
- A 2B VLM at ~1.5 s takes **~300 h** (~12 days of continuous GPU). That's not realistic on a laptop that sleeps and leaves the house.
- So the notes.md hunch holds: rich VLM captioning is for a subset of Segments (top-k re-ranking at query time, or Segments without subtitles), not for the whole library. Florence-2 is the only whole-library captioning candidate.
- Captioner latency is the least certain number in this doc. Measure it first in #9 or a follow-up.

---

## 4. Text embedders for subtitles

CLIP-family text towers are trained on short alt-text and truncate at 64–77 tokens. They're a poor fit for dialogue lines, and BM25 already covers exact quotes (notes.md). A separate sentence embedder is the usual choice.

| Model | Params | Dim (MRL) | Max seq | MTEB Eng v2 Mean (Task) | MTEB Multilingual Mean (Task) | License |
|---|---|---|---|---|---|---|
| **Qwen3-Embedding-0.6B** | 0.6B | 1024 (32–1024) | 32k | 70.70 | 64.33 | Apache-2.0 [[HF][qwen3emb]] |
| **EmbeddingGemma-300M** | 0.3B | 768 (512/256/128) | 2048 | 69.67 (768d), 68.37 (256d) | 61.15 | Gemma terms [[HF][eg1]] |
| EmbeddingGemma-300M QAT Q8_0 / Q4_0 | 0.3B | 768 | 2048 | 69.49 / 69.31 | 60.93 / 60.62 | Gemma terms [[HF][eg1]]. Runs in under 200 MB RAM quantized [[Google blog][eg1-blog]] |
| EmbeddingGemma 2 (text only) | ~270M loaded | 768 | 8k | 68.46 | 61.36 | Apache-2.0 [[card][eg2-card]] |
| BGE-M3 | ~0.6B | 1024, plus sparse and ColBERT multi-vector | 8192 | — | 59.56 (per Qwen's table) | MIT [[HF][bgem3]], [[Qwen table][qwen3emb]] |
| all-MiniLM-L6-v2 | 22.7M | 384 | 256 word pieces | — | — | Apache-2.0 [[HF][minilm]] |

If EmbeddingGemma 2 is picked as the frame embedder, it also covers subtitles in the *same* vector space, which may simplify fusion. Whether one shared space beats two specialist spaces fused with RRF is a question for #9.

---

## 5. Text-encoder cost on CPU (query path on a mini PC)

No primary source benchmarks these *text* towers on a CPU. The estimate below is anchored on the one measured CPU number, from timm on an **Intel i7-12700H** (fp32, batch 1, torch.compile): CLIP B/32 image tower at 8.82 GFLOPs runs in **41 ms** [[timm CPU CSV][timm-cpu]]. That works out to ~215 GFLOP/s effective. Text-tower GFLOPs come from the [OpenCLIP model profile][oc-prof], at full context.

| Text tower | GFLOPs [src] | i7-12700H latency [est] | Low-power mini PC (assumed 2–4× slower) [est] |
|---|---|---|---|
| CLIP B (B/32, B/16, MobileCLIP2-B) | 5.96 | ~30 ms | 60–120 ms |
| SigLIP B/16 | 11.0 | ~50 ms | 100–200 ms |
| CLIP L/14 | 13.3 | ~60 ms | 120–250 ms |
| CLIP H/14 | 47.1 | ~220 ms | 0.45–0.9 s |
| PE-Core L/14 text (0.31B, 24 layers × 1024 wide [[paper][pe-paper]], ≈ the H/14 text class) | ~47 [est] | ~220 ms | 0.45–0.9 s |
| SigLIP So400m text | 53.1 | ~250 ms | 0.5–1 s |
| all-MiniLM-L6-v2 / EmbeddingGemma-300M | — | not measured. Expect ~10 ms / ~50–150 ms [est] | — |

**Takeaways.**
- Every candidate can embed a query on CPU in **about a second or less**. That's viable for a search bar, so the GPU laptop doesn't have to be awake to serve queries. This is good news for ADR-002.
- So400m and PE-Core L text towers sit at the slow end (~0.5–1 s on a weak mini PC [est]). ONNX Runtime plus int8 dynamic quantization would likely cut that 2–4× [est, unverified], but retrieval quality after int8 has to be measured.
- The **query text tower has to be the same model and version** as the indexing image tower. Keeping that model in sync across the GPU indexing path and the CPU query path is a versioning concern (notes.md, "how do they keep the batch and serving paths on the same model?").

---

## Estimation method

- **4060 Laptop throughput** = timm RTX 4090 desktop img/s ÷ 6 to ÷ 8.
  - Basis: NVIDIA lists 1321 AI TOPS and 16,384 CUDA cores for the desktop 4090 [[NVIDIA][nv-4090]], against 233 AI TOPS, 3,072 CUDA cores and a 128-bit bus for the 4060 Laptop [[NVIDIA][nv-laptop]]. That's a ratio of ~5.7×.
  - I widened the range to ÷8 for memory bandwidth and for the 35–115 W laptop TGP range [[NVIDIA][nv-laptop]]: a low-TGP laptop will be slower still.
- **VRAM, weights** = params × 2 bytes (fp16/bf16). Activations at moderate batch sizes are a rough 1–3 GB extra.
- **CPU latency** = GFLOPs ÷ ~215 GFLOP/s effective, derived from the one measured timm CPU row above. The 2–4× slowdown for a mini PC assumes an N100-class or similar low-power chip; the actual mini PC CPUs aren't listed in DESIGN.md D2.
- **Captioner latency** comes from parameter count and the typical tokens generated per caption (~20–30 for short captions, ~100+ for detailed ones). It's the least reliable estimate here.

## Open questions surfaced

1. **Domain gap.** COCO/Flickr scores may not transfer to film frames: dark scenes, letterboxing, stylized grading, animation. #9's eval set should include these.
2. **Single shared space vs. specialist spaces.** One model for subtitles and frames (EmbeddingGemma 2) vs. SigLIP 2/PE for frames plus Qwen3/EmbeddingGemma for subtitles, fused with RRF.
3. **Multi-frame pooling per Segment.** How many frames, mean pooling vs. separate vectors, and whether PE-Core's 8-frame mean helps the motion queries (D16 risk).
4. **Which captioner, if any.** Florence-2 on every Segment vs. a 2B VLM only on top-k at query time (a latency-path cost) vs. no captions at all.
5. **CPU query encoder.** The mini PC CPU model, ONNX/int8 export of the chosen text tower, and the quality loss from quantization.
6. **Decode throughput.** At 100+ frames/s, the GPU likely out-runs keyframe decode over NFS on the mini PCs. That should be measured before optimizing the model.
7. **Model versioning.** Re-embedding the library when the model changes, and keeping the query encoder in sync with it.

---

[oc-res]: https://github.com/mlfoundations/open_clip/blob/main/docs/openclip_retrieval_results.csv
[oc-prof]: https://github.com/mlfoundations/open_clip/blob/main/docs/model_profile.csv
[siglip2]: https://arxiv.org/abs/2502.14786
[siglip2-b]: https://huggingface.co/google/siglip2-base-patch16-256
[siglip2-so]: https://huggingface.co/google/siglip2-so400m-patch16-384
[pe-gh]: https://github.com/facebookresearch/perception_models
[pe-hf]: https://huggingface.co/facebook/PE-Core-L14-336
[pe-paper]: https://arxiv.org/abs/2504.13181
[mobileclip]: https://github.com/apple/ml-mobileclip
[timm-gpu]: https://github.com/huggingface/pytorch-image-models/blob/main/results/benchmark-infer-amp-nchw-pt240-cu124-rtx4090.csv
[timm-cpu]: https://github.com/huggingface/pytorch-image-models/blob/main/results/benchmark-infer-fp32-nchw-pt240-cpu-i7_12700h-dynamo.csv
[eg2-card]: https://gemma.ai/models/embeddinggemma/model-card-v2/
[eg2-paper]: https://arxiv.org/abs/2610.37364
[eg1]: https://huggingface.co/google/embeddinggemma-300m
[eg1-blog]: https://developers.googleblog.com/en/introducing-embeddinggemma/
[videoprism]: https://github.com/google-deepmind/videoprism
[iv2]: https://huggingface.co/OpenGVLab/InternVideo2-Stage2_1B-224p-f4
[florence]: https://huggingface.co/microsoft/Florence-2-large
[moondream]: https://huggingface.co/vikhyatk/moondream2
[smolvlm2]: https://huggingface.co/HuggingFaceTB/SmolVLM2-2.2B-Instruct
[qwen3vl]: https://huggingface.co/Qwen/Qwen3-VL-2B-Instruct
[qwen25vl]: https://huggingface.co/Qwen/Qwen2.5-VL-3B-Instruct
[qwen25vl-lic]: https://huggingface.co/Qwen/Qwen2.5-VL-3B-Instruct/blob/main/LICENSE
[qwen3emb]: https://huggingface.co/Qwen/Qwen3-Embedding-0.6B
[bgem3]: https://huggingface.co/BAAI/bge-m3
[minilm]: https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2
[nv-4090]: https://www.nvidia.com/en-us/geforce/graphics-cards/40-series/rtx-4090/
[nv-laptop]: https://www.nvidia.com/en-us/geforce/laptops/compare/
