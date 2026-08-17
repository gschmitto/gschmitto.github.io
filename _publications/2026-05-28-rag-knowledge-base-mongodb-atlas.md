---
title: "Building a RAG Knowledge Base on MongoDB Atlas"
collection: publications
permalink: /publications/rag-knowledge-base-mongodb-atlas
excerpt: 'Composition, retrieval fidelity, and the failure modes between $vectorSearch and a cited answer. This article walks through the architectural decisions behind a complete RAG knowledge base built on Atlas Search and Atlas Vector Search: chunks as their own collection, static index mappings with filter fields, section-aware chunking, hybrid retrieval with Reciprocal Rank Fusion, reranking as a calibrated truth signal, and the layered defenses that keep a kNN pipeline from answering confidently when the corpus has no answer.'
date: 2026-05-28
venue: 'MongoDB (dev.to)'
paperurl: 'https://dev.to/mongodb/building-a-rag-knowledge-base-on-mongodb-atlas-4jmj'
featuredimage: /images/mongodb-rag-knowledge-base/search-interface.png
featuredimage_alt: "Imagem de cabeçalho"
featuredimage_caption: "Search interface with a streaming AI answer, inline citations, and the source passages that grounded the response."
tags:
  - mongodb
  - rag
  - vector-search
  - ai
---
## Summary

The hard part of a production RAG system is not running a single `$vectorSearch`. It is composing keyword retrieval, vector retrieval, ranking fusion, reranking, and a generative model into a pipeline that returns the right passage, with the right confidence, every time, under concurrency, across tenants, and with predictable latency.

In MongoDB Atlas, this composition can be done end-to-end inside a single cluster. Operational data, full-text indexing, vector indexing, multi-tenant filtering, and the retrieval that feeds an LLM are not separate stores. They are stages of the same aggregation framework.

This text walks through the architectural decisions behind a complete RAG knowledge base built on Atlas Search and Atlas Vector Search, examining where each stage contributes, where each stage fails silently, and what it takes to keep the pipeline honest end-to-end.

## Composition, not capability

A knowledge base in production exposes a single search box. Behind that box, there are five distinct stages of execution. Treating them as a single primitive ("the LLM answers questions about our docs") is the most common reason these systems return confidently wrong answers.

The actual pipeline looks like this:

![The five stages of the RAG pipeline on MongoDB Atlas](/images/mongodb-rag-knowledge-base/pipeline-stages.png)

The first four stages are [MongoDB-native](https://www.mongodb.com/products/platform/atlas-search/). The fifth is the only component outside Atlas, and it consumes context retrieved entirely from MongoDB. Each transition between stages has a cost and a contract. Understanding those contracts is what separates a demo from a production system.

The remainder of this article examines those contracts: data layout, index design, ranking, the role of reranking as a truth signal, and the failure mode that vector search shares with every nearest-neighbor algorithm: the one where it never tells you it found nothing.

## Data layout: chunks as their own collection

The first non-trivial decision in any [RAG system on MongoDB](https://www.mongodb.com/docs/vector-search/tutorials/rag/) is whether to store chunks as an embedded array inside the parent article, or as documents in their own collection.

Embedded arrays look attractive. They preserve locality, the document hierarchy mirrors the application model, and a single `findOne` returns everything. They are also the wrong choice for non-trivial corpora.

Three reasons argue for a separate chunks collection.

First, **retrieval granularity**. `$vectorSearch` returns the documents it scored, not their parents. With chunks embedded inside articles, a vector hit identifies the article but not the passage. The user receives a long document containing the answer somewhere; the system has lost the ability to point at the exact section. With chunks as documents, each hit carries its own `_id`, its position in the article (`chunkIndex`), and the precise text that scored.

Second, **re-indexing cost**. Updating one article when chunks are embedded means rewriting the entire parent document, including every untouched chunk and its embedding. With chunks in their own collection, re-indexing is a `deleteMany({ articleId })` followed by an `insertMany` of the new chunks. The articles collection remains compact and serves the operational queries of the application (list, filter, render) without dragging embedding payloads through every read.

Third, and most consequentially, **pre-filter at index time**. The vector search index can pre-filter candidates inside `$vectorSearch` itself, before any document is hydrated, but only on top-level fields of the indexed collection. When chunks live as documents, denormalizing `workspaceId`, `visibility`, `tags`, and `category` onto each chunk turns multi-tenancy from a downstream `$match` into an index-level prune. The HNSW traversal walks only the candidate set that the tenant is allowed to see. This is the difference between a system that scales linearly with the corpus and one that scales with the per-tenant slice.

```js
chunks: {
  _id:         ObjectId,
  articleId:   ObjectId,
  workspaceId: string,
  chunkIndex:  Number,
  text:        String,
  embedding:   [Number],   // 1024 floats from voyage-3
  tags:        [String],
  category:    String,
  visibility:  String,
}
```

One detail of the denormalization is worth calling out. `workspaceId` is stored on chunks as a string, not as an `ObjectId`, because Atlas Vector Search filter fields do not compare `ObjectId` values directly. The chunk contains the hex representation of the parent workspace's `_id`, which is what the `$vectorSearch` filter clause compares against. Articles that are not filtered through a vector index continue to use the native `ObjectId` type.

## Index design: static mappings and filter fields

Two Atlas indexes handle retrieval, one per collection. Both use static mappings. Dynamic mappings index every field of every document, including fields that never appear in queries, which inflates the index and slows every query. Static mappings force every indexed field to justify its existence.

The full-text index on articles declares language analyzers per field:

```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title":       { "type": "string", "analyzer": "lucene.english" },
      "content":     { "type": "string", "analyzer": "lucene.english" },
      "summary":     { "type": "string", "analyzer": "lucene.english" },
      "tags":        { "type": "string", "analyzer": "lucene.keyword" },
      "category":    { "type": "string", "analyzer": "lucene.keyword" },
      "workspaceId": { "type": "objectId" }
    }
  }
}
```

`title`, `content`, and `summary` use the English analyzer (lowercasing, stemming, stopword removal) so that natural-language queries match natural-language text. `tags` and `category` use the keyword analyzer for exact-match behavior without lexical transformation. `workspaceId` is indexed as an `ObjectId`, so tenant scoping is exact.

The vector index on chunks declares the embedding geometry and the fields eligible for pre-filtering inside `$vectorSearch`:

```json
{
  "fields": [
    { "type": "vector", "path": "embedding",
      "numDimensions": 1024, "similarity": "cosine" },
    { "type": "filter", "path": "workspaceId" },
    { "type": "filter", "path": "visibility" },
    { "type": "filter", "path": "tags" },
    { "type": "filter", "path": "category" }
  ]
}
```

**`numDimensions: 1024`** matches the output dimensionality of the chosen embedding model. Cosine similarity is the standard metric for normalized text embeddings. The four filter declarations are not metadata. They are the difference between a vector index that scopes to a single workspace inside HNSW traversal and one that scans the entire corpus before filtering downstream. At any non-trivial corpus size, the first behaves and the second does not.

## Section-aware chunking and breadcrumb context

Chunking decides retrieval quality before any embedding model is involved. A 1024-dimensional vector is a single point representing an entire chunk of text. If that text contains three unrelated paragraphs, the resulting embedding sits in the centroid of three semantic regions and is meaningfully close to none of them.

Fixed-window chunkers ("every 800 characters with 120 character overlap") produce exactly this problem. They cut mid-sentence, mid-section, and mid-topic, and the embedding is a faithful representation of the resulting incoherence.

Section-aware chunking solves this at the document level. ATX headings (`#`, `##`, `###`) demarcate semantic units that the author has already labeled. Splitting along those boundaries produces chunks that map to a single logical idea each, and the embedding represents that idea with no contamination from neighboring sections.

Two additional properties matter at scale.

The **breadcrumb path** is prepended to every chunk of text:

```text
Designing a chunking strategy for RAG > Semantic chunking by section
The most precise approach for structured documents is to chunk by heading: every ## or ### section becomes one chunk…
```

The embedding now encodes hierarchical context, not just local content. When the user asks about a specific topic, the model sees the chunk's position in the document graph and weights it accordingly. The reranker sees the same context, and the LLM, in the final stage, receives a passage with a self-identifying header, which materially improves citation quality.

The fallback for oversized sections is unconditional. A section larger than the maximum chunk size (1500 characters in this implementation) falls back to paragraph splitting, then to sentence splitting, while the breadcrumb is preserved on every sub-chunk. The system never produces a chunk without context.

Fenced code blocks are excluded from heading detection. A `#` comment inside a bash block is content, not structure, and treating it as a heading would cause chunks to break unpredictably.

## Hybrid retrieval and the score-scale problem

Each retrieval stage is a single aggregation pipeline. The keyword stage ranks articles by BM25, with fuzzy tolerance and per-field boosts:

```js
db.articles.aggregate([
  {
    $search: {
      index: "articles_search",
      compound: {
        should: [
          { text:  { query, path: "title", fuzzy: { maxEdits: 1 } },
            score: { boost: { value: 3 } } },
          { text:  { query, path: "content", fuzzy: { maxEdits: 1 } } },
          { text:  { query, path: "tags" },
            score: { boost: { value: 2 } } },
        ],
      },
    },
  },
  { $match: { workspaceId: ObjectId("...") } },
  { $limit: 30 },
  { $project: { _id: 1, score: { $meta: "searchScore" } } },
])
```

The semantic stage runs in parallel, on the chunks collection, with the workspace filter applied inside the index:

```js
db.chunks.aggregate([
  {
    $vectorSearch: {
      index: "chunks_vector",
      path: "embedding",
      queryVector: [/* 1024 floats from Voyage */],
      numCandidates: 200,
      limit: 50,
      filter: { workspaceId: "65f1b…" }
    },
  },
  { $project: {
      articleId: 1, chunkIndex: 1, text: 1,
      score: { $meta: "vectorSearchScore" } } },
])
```

Both stages produce ranked result lists, but on incomparable scales.

Atlas Search returns BM25-style scores that depend on term frequency, inverse document frequency, and document length. Scores routinely climb into the tens and can spike even higher when a document is short, and the queried term is densely used.

Atlas Vector Search returns cosine similarities bounded between 0 and 1, with realistic values for relevant matches typically clustering in 0.6 to 0.9.

Adding these scores, or normalizing them to a common scale, is fragile. A handful of documents with anomalously high BM25 scores will dominate the fused ranking regardless of their actual relevance. Linear weighting with hand-tuned coefficients works for one corpus and fails on the next.

**Reciprocal Rank Fusion** sidesteps the problem by discarding scores entirely and working only on rank position:

```text
rrf_score(doc) = Σ over rankers   1 / (k + rank_i(doc))     // k = 60
```

A document at rank 1 contributes 1/61 per ranker, rank 2 contributes 1/62, and so on. The scale collapses. Documents that appear near the top of multiple rankers accumulate score; documents that appear near the top of only one ranker also score, but less. The fusion is parameter-free in practice: k = 60 is the value from the original 2009 paper and remains the production default.

In this implementation, fusion is anchored on chunks. Semantic search provides the candidate list; keyword search acts as a boost when a chunk's parent article also matches the keyword. This asymmetry is intentional: expanding every keyword-matched article into all of its chunks would pollute the candidate pool with off-topic content from articles that matched on a single tangential term.

```js
const fused = semanticChunks
  .map((chunk, idx) => {
    let score = 1 / (k + idx + 1);                              // semantic rank
    const kwRank = keywordArticleRank.get(chunk.articleId.toString());
    if (kwRank !== undefined) score += 1 / (k + kwRank + 1);    // keyword boost
    return { ...chunk, rrfScore: score };
  })
  .sort((a, b) => b.rrfScore - a.rrfScore)
  .slice(0, 20);
```

[MongoDB 8.1 introduced `$rankFusion`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/rankFusion/) as a first-class aggregation stage that performs RRF natively. Production code should prefer it; the explicit implementation above exists to make the algorithm visible.

## Reranking as the truth signal

After fusion, the system has a ranked list of twenty chunks, but no calibration of how good that ranking actually is. A list of twenty chunks always exists, even when none of them is relevant.

Reranking solves two problems simultaneously. First, it improves precision: the top of the fused list is reordered by a cross-encoder model that scores each (query, passage) pair directly, which is materially more accurate than approximate nearest-neighbor distance for the small set of candidates that matters. Second, and more critically, the reranker's score becomes a calibrated relevance signal that the rest of the system can interpret.

Voyage's **rerank-2** model, exposed through the MongoDB-managed gateway at **ai.mongodb.com/v1**, returns scores in the **[0, 1]** range. Empirically, on a well-formed corpus:

- A top score **above 0.5** indicates strong relevance: the passage genuinely contains the answer.
- A top score **between 0.3 and 0.5** indicates partial relevance: adjacent topics, paraphrases, weak matches.
- A top score **below 0.3** indicates the corpus does not contain an answer to the question.

This calibration is what makes the downstream stages safe. The LLM receives passages with known quality. The UI can surface a low-confidence banner. The application can decide at query time whether to attempt a generated answer.

The reranker is more expensive per pair than the embedding model, which is why it runs only on the shortlist produced by fusion. The cost is linear in the number of candidates reranked, and reranking twenty candidates is small even at high concurrency.

## The generative stage as a constrained consumer

The LLM does not "answer questions about the knowledge base". It receives a prompt containing retrieved passages and is instructed, by system message, to use only those passages.

The contract is explicit, and the model honors it:

```plaintext
SYSTEM:
You are a helpful assistant answering questions using ONLY the passages
provided below. Cite the passages you used as bracketed numbers like [1]
or [2]. If the passages do not contain enough information to answer, say
so honestly. Reply in the same language as the user's question.
USER:
Passages:
[1] Designing a chunking strategy for RAG > Semantic chunking by section
    The most precise approach for structured docs is to chunk by heading...
[2] Designing a chunking strategy for RAG > Reasonable defaults
    A starting point is 500-1500 characters per chunk...
[3] Getting started with MongoDB Atlas Vector Search > The chunking pattern
    A common pattern is to chunk long documents, embed each chunk...
Question: How do I chunk by markdown section?
```

What returns is the text that the model generated by reading those passages. The citation discipline (`[1]`, `[2]`) is enforced by the prompt, parsed by the frontend, and rendered as in-text links that scroll to the source card. The frontend never displays an unreferenced answer alongside hidden sources, and the model never sees passages it isn't allowed to cite.

Streaming changes the user experience but not the architecture. The response is delivered as newline-delimited JSON, with three event types:

```json
{"event":"passages","passages":[...]}     // emitted once, before the LLM call
{"event":"token","text":"..."}            // emitted per token
{"event":"done","model":"gpt-4o-mini"}    // emitted at the end
```

Passages are sent first, before the LLM call begins. The frontend renders source cards immediately and updates the answer panel as tokens arrive. From the user's perspective, the system is already useful before the model has finished generating.

![The AI answer card with inline numbered citations linking to the source passages retrieved from MongoDB](/images/mongodb-rag-knowledge-base/ai-answer-citations.png)
*The AI answer card with inline numbered citations linking to the source passages retrieved from MongoDB.*

## The failure mode: $vectorSearch never says "no"

Nearest-neighbor search is by construction non-empty. Given a query vector, the HNSW graph returns the N closest vectors regardless of whether those vectors are actually relevant to the query. A query about the capital of a country, posed to a knowledge base about MongoDB, will receive five chunks ranked by cosine distance: chunks that share no semantic content with the question.

This is not a bug. It is the contract of kNN. The pipeline must compensate at the application layer.

![Low-confidence banner shown when the top reranker score falls below the relevance threshold](/images/mongodb-rag-knowledge-base/low-confidence-banner.png)
*Low-confidence banner shown when the top reranker score falls below the relevance threshold; no sources are surfaced.*

Three layered defenses keep the system honest.

The **system prompt** instructs the LLM to refuse questions whose passages do not contain the answer. The model honors this when scores are uniformly low; it produces "the passages provided do not contain information about that" rather than hallucinating an answer from training data.

The **rerank score** is the truth signal. When the top reranker score falls below the strong-relevance threshold, the system knows the corpus is not coherent with the question. This is the only stage in the pipeline that produces calibrated relevance. Vector distance and BM25 scores cannot be interpreted in isolation, but rerank scores can.

The **UI threshold** makes the calibration visible. Below a confidence threshold (empirically 0.4 for the top rerank), the interface displays a banner indicating that the knowledge base does not cover the topic, and hides individual passages whose scores fall below a stricter threshold (empirically 0.3). The user does not see five low-quality "sources" that contradict the LLM's refusal.

```js
const STRONG_RELEVANCE = 0.4;
const WEAK_PASSAGE     = 0.3;

const confidence = passages[0].rerankScore >= STRONG_RELEVANCE ? "strong" : "weak";
const visiblePassages = passages.filter(p => p.rerankScore >= WEAK_PASSAGE);
```

Each defense alone is insufficient. The prompt can fail when scores happen to land in the partial-relevance band. The threshold can hide useful passages in rare edge cases. The UI banner alone cannot prevent a confident hallucination. Together, they make confidently wrong answers very unlikely.

## Conclusion

A production RAG system is the composition of five well-understood stages, each with its own contract and its own failure modes. The interesting engineering does not live in any single stage. It lives at the boundaries between them: in the data model that makes pre-filtering possible, in the chunking that makes embeddings coherent, in the fusion that reconciles incomparable scores, in the rerank that produces a calibrated truth signal, and in the prompt that translates that signal into a refusal when the corpus does not have the answer.

[MongoDB Atlas](https://www.mongodb.com/products/platform/atlas-database/) is the substrate that lets these stages compose without crossing infrastructure boundaries. The operational data, the search indexes, and the vector indexes are not federated systems with their own consistency models. They are aggregation stages in the same query language, against the same documents, with the same access control. The cost saved is not raw latency, which is a property of every well-engineered system; it is the entire class of consistency bugs that arises when a document, a search index, and a vector store have to agree about the same change.

RAG is not a prompt problem. It is a systems engineering problem. When the boundaries are honest, the answers are honest.
