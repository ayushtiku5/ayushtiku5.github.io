---
layout: post
title: "Building an Agentic RAG Assistant on Google's ADK"
date: 2026-07-25 09:00:00 -0000
categories: ai
description: "An agentic RAG assistant on Google's ADK that decides when to search, rephrase, or decompose questions instead of running a fixed retrieval pipeline."
---

Most RAG demos do one retrieval call per question and call it done. That works for simple lookups, but it falls apart the moment a question is ambiguous or has multiple parts. I wanted to see what RAG looks like when the retrieval step is a tool the model chooses to call, rather than a fixed step that always runs before generation. Code is here: [github.com/ayushtiku5/agentic-rag](https://github.com/ayushtiku5/agentic-rag).

It's built on Google's Agent Development Kit (ADK), backed by a local ChromaDB vector store, with the LLM behind it swapped to Claude via ADK's `AnthropicLlm` model adapter.

## The shape of it: four tools, one instruction block

The agent itself is a short declaration. All the actual behavior comes from the tools it's given and the instructions describing how to use them:

```python
root_agent = Agent(
    name="rag_agent",
    model=AnthropicLlm(model="claude-sonnet-4-6"),
    tools=[ingest_document, search_documents, list_documents, delete_document],
    instruction=(
        "1. Always call search_documents before answering any factual question.\n"
        "2. If the first search returns weak or insufficient context, rephrase the "
        "   query and search again (up to 2 additional attempts).\n"
        "3. For complex, multi-part questions, decompose them into sub-questions "
        "   and run a separate search for each part.\n"
        ...
    ),
)
```

That's the "agentic" part of agentic RAG. There's no orchestration code that forces a search before every response. The model decides to search, decides whether the results are good enough, and decides whether to retry with a rephrased query or split a multi-part question into sub-searches. The instruction block is the only thing steering that behavior, which means the quality of the retrieval strategy lives in prompt engineering, not control flow.

## Ingestion: chunk, embed, store

Getting text into the store is a three-step pipeline in `ingestion.py`, kept deliberately plain:

```python
def _chunk_text(text: str, chunk_size: int = 1000, overlap: int = 200) -> list[str]:
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start += chunk_size - overlap
    return [c for c in chunks if c.strip()]
```

Fixed-size character chunks with a 200-character overlap, so a sentence that happens to fall on a chunk boundary still shows up whole in at least one neighboring chunk. Nothing sentence-aware or semantic about the split, just a sliding window. `.txt`, `.md`, `.pdf` (via `pypdf`), and `.docx` (via `python-docx`) all funnel through the same chunker after their text is extracted.

Each chunk gets an ID derived from a hash of its source, index, and first 64 characters, which makes re-ingesting the same file idempotent: `vector_store.add_documents` diffs incoming chunk IDs against what ChromaDB already has and only embeds the new ones.

```python
def add_documents(chunks: list[dict]) -> None:
    collection = _get_collection()
    existing_ids = set(collection.get(ids=[c["id"] for c in chunks])["ids"])
    new_chunks = [c for c in chunks if c["id"] not in existing_ids]
    if not new_chunks:
        return
    embeddings = embed_texts([c["text"] for c in new_chunks])
    collection.add(...)
```

Embeddings come from a local `sentence-transformers` model (`all-MiniLM-L6-v2`), not an API call, so ingesting a large document doesn't cost anything beyond CPU time and doesn't leak document content to a third party.

## Retrieval: cosine similarity, nothing fancier

Search is a straight nearest-neighbor query against the ChromaDB collection, configured for cosine distance:

```python
_collection = _client.get_or_create_collection(
    name=_COLLECTION_NAME,
    metadata={"hnsw:space": "cosine"},
)
```

`search_documents` wraps this and converts ChromaDB's distance metric into something more legible for the model to reason about: `relevance_score = 1.0 - distance`, so higher is better instead of lower. It also caps `n_results` between 1 and 20 regardless of what the model asks for, a small guard against a malformed or runaway tool call.

No reranking, no hybrid keyword-plus-vector search, no query expansion beyond what the model itself does by rephrasing and calling the tool again. Those are the obvious next steps if retrieval quality turns out to be the bottleneck, but they weren't needed to get a working agentic loop going.

## Where the "agentic" part actually earns its keep

The interesting failure mode this design handles well is a compound question like "what does the architecture doc say about caching, and how does that compare to what the deployment guide says about scaling." A fixed one-shot RAG pipeline embeds that whole question, does one search, and gets back chunks that are mediocre matches for both halves. Because the instruction tells the model to decompose multi-part questions, it instead calls `search_documents` twice, once per sub-question, and gets two sets of chunks that are each a strong match for their half.

That only works because search is a tool call the model makes deliberately, with the results fed back into the same conversation before it commits to an answer, rather than a retrieval step baked into a fixed pipeline before generation ever starts.

## What's not here

There's no reranking model, no query rewriting beyond what the system prompt asks the LLM to do on its own, and no evaluation harness to measure whether the "search again if results are weak" instruction actually improves answer quality versus just adding noise. The vector store is a single local ChromaDB collection with no metadata filtering beyond source and chunk index, so there's no way to scope a search to, say, only PDFs or only documents ingested in the last week. All reasonable next steps, but the core loop, model decides to search, evaluates what comes back, and decides whether to search again, was the part worth proving out first.
