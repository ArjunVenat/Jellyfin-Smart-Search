# Research Notes

Working notes and brainstorming. This is **not** the design. When a note turns into a decision, write an ADR in [arch/](arch/) and add a row to D15 in [DESIGN.md](DESIGN.md).

<!-- The "Related decision" tags point to rows in DESIGN.md D15, so it's clear which decision a note feeds. -->

---

## Retrieval approach
*Related decision: overall retrieval approach ([ADR-001](arch/ADR-001-multimodal-rag-pipeline.md))*

Multimodal RAG: scenes and subtitles are separate sources.

BM25 + vector search with RRF.

BM25 is a good fit for text, especially for searching direct quotes or similar dialogue.

Vector search aims to capture the meaning of images and find similar ones.

In multimodal RAG, the retriever pulls text and images based on the query (subtitles and specific frames). This needs a multimodal embedding model that can embed both images and text. Example with CLIP:

1. Generate model-specific embeddings for all data, both text and images, and store them in a vector DB.
2. Given a query, generate its model-specific embedding.
3. Query the vector DB for all images and texts whose embeddings are close to the query embedding.

Graph RAG is useful for connections between things; vector RAG is good for finding similar things.

Query rewriting is not needed, because searches are stateless and independent of each other: the user types into a search bar rather than talking to an LLM.

Contextual retrieval: useful for attaching context to each chunk so retrieval is easier, possibly using tags, keywords, or prompting an AI to augment the chunk with its original context.

## Chunking / search unit
*Related decision: search unit (add to D15)*

Chunk on semantic boundaries: a single shot or scene, with 15 s sampling for one-take sequences.

Maybe use PySceneDetect, though it may be CPU-bound?

## Models
*Related decision: image embedder / captioner (add to D15)*

What model to use? LLaVA? SigLIP? CLIP?

LLaVA is useful for detailed captioning, but that richness costs speed. Not every frame can be captioned: there are hundreds of movies and thousands of episodes.

## Throughput vs. latency
*Related decision: [ADR-002](arch/ADR-002-throughput-vs-latency.md)*

*DDIA*, *AI Engineering* and *Inference Engineering* agree that throughput and latency should be optimized separately. Here, throughput = offline batch processing of media; latency = each query.

To research: real-world offline batch media processing. Leads to check (unverified, from memory):
- Netflix Tech Blog: "Building In-Video Search": searching for moments inside video with embeddings. The closest real-world match to this project.
- Netflix Tech Blog: "Match Cutting at Netflix": shot detection plus embeddings over a catalog.
- Netflix Tech Blog: "Scaling Media Machine Learning at Netflix": batch media ML infrastructure vs. online serving.
- Netflix Tech Blog: the Cosmos platform: media processing as queued, resumable workflows.
- Batch vs. online feature/inference split in ML platforms in general (e.g. Uber's Michelangelo).

Questions to bring to the reading: How do they make jobs resumable and idempotent? How do they version derived data when a model changes? How do they keep the batch and serving paths on the same model?

## Evaluation
*Related section: DESIGN.md D5*

Golden data set / evals:

LLM as judge.

---

## Open questions to investigate
<!-- Questions you haven't answered yet. Move them into D15 rows once they're real decisions. -->
