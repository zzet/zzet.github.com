---
title: "Graph RAG vs vector RAG for code"
description: "Graph RAG vs vector RAG for code: an honest, code-specific breakdown of embeddings, resolved edges, why agents reach for grep, and the hybrid RRF answer."
date: 2026-05-27
lastmod: 2026-06-06
draft: false
slug: "graph-rag-vs-vector-rag-for-code"
keywords: ["graph RAG vs vector RAG for code", "knowledge graph vs vector database for code RAG", "why AI agents use grep not vectors", "deterministic code retrieval vs embeddings", "hybrid code retrieval rrf", "graph rag for code", "vector rag for code", "reciprocal rank fusion code search"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You have heard both memes. One camp says *RAG is dead for code* — agents should just grep, the way Claude Code and Cursor seem to. The other camp says *GraphRAG changes everything* — resolved edges beat embeddings. Both are oversold, and if you are choosing how to feed a coding agent context, the slogans will steer you wrong.

So here is the honest version of **graph RAG vs vector RAG for code**, for someone who has to actually pick. Vector RAG gives strong recall on fuzzy natural-language questions but is approximate, structure-blind, and goes stale the moment you edit. Graph RAG is deterministic and answers structural questions — blast radius, call chains — but it needs parsing and symbol resolution up front. Neither wins outright. The position that survives contact with a real codebase is *hybrid*.

{{< lead >}}Graph RAG vs vector RAG for code is not a contest with a winner. Vector RAG embeds code chunks and ranks by similarity — great recall on fuzzy "find code that does X" queries, but approximate, structure-blind, and stale after every edit. Graph RAG stores resolved edges (calls, implements, references) and answers structural questions deterministically, at the cost of upfront parsing. The honest answer is to fuse them: lexical (BM25) + semantic (vectors) + graph-centrality, blended with Reciprocal Rank Fusion. Pure-vector tools can still win on purely semantic recall at scale.{{< /lead >}}

This article is part of the broader case for [a code knowledge graph for LLM agents](/gortex/code-knowledge-graph-for-llms/). Read the definition there first if "resolved edge" is new to you; here we focus on the retrieval trade-off specifically.

---

## What vector RAG for code actually is

Vector RAG has a simple pipeline. Split the codebase into chunks, embed each chunk into a vector, store the vectors in an index, then at query time embed the question and return the nearest neighbors by cosine similarity (approximate nearest-neighbor, or ANN, search). It is the machinery of semantic document search, pointed at source files.

This works genuinely well for one shape of question: fuzzy, natural-language intent. *"Where do we validate JWT expiry?"* has no single keyword to grep, but the surrounding code reads semantically close to the query, so the embedding finds it. That is the real strength of vector RAG, and it is not small.

The best code-RAG systems lean into it hard. [Qodo's RAG for large-scale code repos](https://www.qodo.ai/blog/rag-for-large-scale-code-repos/) chunks at roughly 500 characters with static analysis, re-adds the enclosing class and imports, embeds an LLM-generated description alongside the code, then does two-stage retrieval (vector similarity, then an LLM rerank). Their open [Qodo-Embed-1 model beats OpenAI and Salesforce](https://venturebeat.com/programming-development/qodos-open-code-embedding-model-sets-new-enterprise-standard-beating-openai-salesforce) on the CoIR benchmark. Concede it plainly: careful chunking plus a strong code-embedding model goes a long way on semantic recall.

### The three failure modes of vector RAG on code

The trouble is that code is not prose, and embeddings inherit three problems when you treat it like prose.

| Failure mode | What happens |
|---|---|
| **Approximate** | ANN returns what *looks* similar, ranked by a learned distance. It is a probability, not a fact. The transitive caller two hops away rarely *looks* similar to the callee, so it is missed. |
| **Structure-blind** | "What depends on this module?" has no obvious embedding neighbor. Structural relationships are not encoded in token similarity, so similarity search cannot answer them. |
| **Chunk boundaries** | A function spans multiple chunks; the class it belongs to is in another chunk; the interface it implements is elsewhere. Character or paragraph chunking splits declarations and orphans braces. |

The boundary problem is the one that bites quietly. As the chunking literature documents, a chunk rarely contains a complete logical unit, so retrieval returns fragments that are individually plausible and collectively incoherent. [FalkorDB frames the deeper issue well](https://www.falkordb.com/blog/beyond-vector-search-why-a-code-graph-is-the-secret-to-chatting-with-complex-codebases/): vector search returns "code that looks similar, not code that is actually connected." For a question about connectivity, similarity is the wrong axis.

### The stale-on-edit problem

There is a fourth cost that is operational rather than relevance-shaped: **embeddings go stale on edit**. Every time you change a function, the chunk's embedding is wrong until you re-embed it. For an actively developed repo, keeping the vector index current is its own pipeline — change detection, re-chunking, re-embedding, re-syncing.

[Cursor's own description of secure codebase indexing](https://cursor.com/blog/secure-codebase-indexing) shows how much engineering this demands: a Merkle tree of file hashes for change detection, tree-sitter-aware chunking of only the changed files, async embedding generation, content-addressed caching so only changed chunks re-sync, and a remote vector database holding embeddings but not raw source. That is a well-built freshness pipeline — and the fact that it has to exist at all is the point. We compare these designs in [how Cursor indexes your codebase, embeddings vs a structural graph](/gortex/how-cursor-indexes-codebase-embeddings-vs-graph/).

---

## Why AI agents reach for grep, not vectors

If embeddings are so capable, why do today's coding agents lean so heavily on agentic, grep-style search instead? Two reasons, and naming them precisely matters because they are exactly what a good graph fixes.

**Determinism.** Grep returns exact matches. A code identifier is a keyword that needs an exact match — `parseToken` means `parseToken`, not "something that reads semantically close to token parsing." Grep also fails *loudly*: no match means no match, not a confidently-wrong nearest neighbor. As [Sara Zan argues in "is grep really better than a vector DB"](https://www.zansara.dev/posts/2026-03-15-vector-dbs-vs-grep/), this loud-failure property is underrated — a wrong-but-plausible result is more dangerous than an empty one.

**Freshness.** Grep reads the current file off disk. There is no index to go stale, no embedding to re-sync. The agent always sees the code as it is right now.

That is why the "agents use grep" observation rings true — though it is *inferred from observable behavior*, not a sourced quote from those tools' engineers; the [MindStudio writeup on what AI agents use instead of RAG](https://www.mindstudio.ai/blog/is-rag-dead-what-ai-agents-use-instead) makes the same inference.

But grep has the same blind spot as vectors, from the other direction: **grep is structure-blind too.** It cannot return a resolved call edge or tell you who implements an interface. It greps a name and hands back every textual hit — comments, strings, a different symbol with the same name — and the agent guesses. That false-positive tax is the subject of [finding usages without grep's false positives](/gortex/find-usages-without-grep-false-positives/). Determinism and freshness are necessary; they are not sufficient.

---

## Graph RAG vs vector RAG for code: what the graph adds

Graph RAG replaces text matching with traversal of *resolved edges*. You parse the code, resolve symbols (this `parseToken` call binds to *that* declaration), and store the relationships as a typed graph: `calls`, `implements`, `references`, `imports`, dataflow. Now "who calls this?" is a graph walk that returns verified callers, and "what breaks if I change this?" is a reachability query over the call graph.

This is what grep and vectors both cannot do. A general [comparison of graph RAG and vector RAG](https://www.instaclustr.com/education/retrieval-augmented-generation/graph-rag-vs-vector-rag-3-differences-pros-and-cons-and-how-to-choose/) found vector retrieval better at locating specific documents (54% vs 35%) but graph retrieval roughly 3× better on aggregation queries and 4× better on cross-document reasoning. For code, "cross-document reasoning" *is* the daily job: a call chain crosses files, a blast radius crosses packages.

The honest cost: graph RAG needs the parsing-and-resolution step up front, and resolution is harder than chunking. But it is work done once and patched incrementally, not paid per query.

| | Vector RAG | Grep / agentic search | Graph RAG |
|---|---|---|---|
| Best at | fuzzy NL intent | exact identifiers | structural questions (callers, blast radius) |
| Result type | approximate, ranked | exact, unranked | deterministic, resolved |
| Structure-aware | no | no | yes |
| Fresh after edit | needs re-embed | always | needs incremental patch |
| Setup cost | chunk + embed pipeline | none | parse + resolve up front |

Read that table honestly and the conclusion writes itself: every column wins at something, and none wins at everything.

---

## The honest answer: hybrid retrieval with Reciprocal Rank Fusion

You do not have to choose. The position that holds up is to run all three retrievers and *fuse* their results. Even FalkorDB, which sells a graph engine, recommends a hybrid: use embeddings to find a starting node by intent, then let the graph walk the relationships. HybridRAG consistently beats either approach alone.

The mechanism that makes fusion practical is **Reciprocal Rank Fusion (RRF)**. The problem it solves: BM25 scores are unbounded and cosine similarities sit in roughly [0, 1] — incomparable scales, so you cannot just add them. RRF sidesteps this by ignoring raw scores and using only *rank positions*:

```text
score(doc) = sum over each ranked list of  1 / (k + rank)
default k = 60
```

Each retriever (lexical, vector, graph) ranks documents independently; RRF sums the reciprocal ranks across lists. A result that places near the top in two of three lists rises; a result that only one retriever loves does not dominate. Because it uses rank, not score, BM25 and cosine fuse cleanly. As [OpenSearch's writeup on Reciprocal Rank Fusion](https://opensearch.org/blog/introducing-reciprocal-rank-fusion-hybrid-search/) documents, hybrid RRF yields consistently higher NDCG than lexical or vector alone — and [Azure AI Search uses RRF](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking) as its default hybrid scorer for the same reason.

Two refinements turn RRF from "merge two lists" into a code-specific retriever:

- **Graph-centrality reranking** adds a *third* signal beyond lexical and semantic. After fusion, rerank by structural importance — Random-Walk-with-Restart / Personalized PageRank floats the symbols that are central to the call graph. A widely-called function ranks above an obscure one with the same textual match.
- **Per-query alpha** tunes the lexical-vs-semantic mix continuously. An identifier-shaped query (`parseToken`) leans lexical; a natural-language query (*"where do we validate expiry"*) leans semantic. One knob, set per query, instead of a fixed blend.

---

## Where Gortex fits: graph + vector + lexical, fused

[Gortex](https://github.com/zzet/gortex) is built as the hybrid this section describes, not as "graph instead of vectors." Its retrieval stack is the fusion above, implemented end to end:

| Layer | What it does |
|---|---|
| **Lexical** | BM25 / FTS5 trigram search over symbols, camelCase-aware. |
| **Semantic** | Default-on embeddings, baked **GloVe-50d (3.8 MB embedded)**, zero deps and no API key; opt-in MiniLM / Ollama / OpenAI for stronger recall. |
| **Fusion** | Reciprocal Rank Fusion blends BM25 + vector; a per-query **alpha** tunes the lexical↔semantic mix. |
| **Graph rerank** | Graph-centrality reranking (Random-Walk-with-Restart / Personalized PageRank) floats structurally-central results up. |

It directly answers the two reasons agents reach for grep:

- **Determinism** — the graph stores *resolved* `calls` / `implements` / `references` edges. `find_usages` returns every reference with its context and zero false positives; `get_callers` and `get_call_chain` traverse the call graph; `explain_change_impact` gives a risk-tiered blast radius. These are verified facts, not approximations.
- **Freshness** — fsnotify watches the tree and applies **surgical per-file graph patches** (not a full re-index), incremental re-index ~200 ms vs a 3–5 s full pass. Retrieval does not go stale on edit, so there is no re-embed pipeline to babysit.

Architecturally it is in-process and in-memory with zero external dependencies — no separate graph database server (FalkorDB's model), no remote vector database (Cursor's model), no model download to start. It is **Apache 2.0** licensed, spans **257 languages**, and exposes **100+ MCP tools**. On exact-tier symbol queries the BM25 ranker hits R@5 ≈ 96.8% (3.2× ripgrep's floor), and the token win runs 3–50× fewer tokens per response than naive file reads (peak ~50× on identifier lookups). It ships an eval harness (`gortex eval pack`) so you can score P@K / R@K / MRR on your own repo.

```bash
# index, then hybrid search over the fused stack
gortex index .
gortex query symbol "where do we validate token expiry"   # leans semantic
gortex query symbol "parseToken"                            # leans lexical
```

### The honest limits

The same discipline applies to Gortex. Its **default GloVe-50d embeddings are lightweight** — zero-setup and no API key, but not state-of-the-art semantic recall; for the best concept-level recall you opt into MiniLM or OpenAI. Its strength is the *exact* tier (96.8% R@5) and the deterministic graph; on concept and multi-hop tiers semantic recall is modest (R@5 ≈ 25–30%). The differentiator is *hybrid + deterministic graph*, not semantic supremacy. A few refactor operations (`move_symbol`, `inline_symbol`) are Go-only for now. And it is a binary plus daemon — heavier to bootstrap than a browser/WASM tool, with no in-browser story. Benchmarks are single-machine numbers; absolute timings vary 2–5× by hardware, though reproducible with `gortex bench`.

### Treating the competition fairly

Each of the leading tools is genuinely strong at something:

- **[FalkorDB](https://www.falkordb.com/blog/beyond-vector-search-why-a-code-graph-is-the-secret-to-chatting-with-complex-codebases/)** ships a fast graph engine (GraphBLAS, OpenCypher) and takes an honest hybrid stance — embeddings to find the start node, then walk the graph. Trade-offs: its [code-graph demo](https://docs.falkordb.com/genai-tools/code-graph.html) supports Python, Java, and C# only (as of 2026-06), needs a separate graph-DB server, and is SSPLv1-licensed (not permissive) as of 2026-06.
- **[Glean](https://www.glean.com/product/enterprise-graph)** does enterprise-scale, permissions-aware hybrid search over 100+ connectors (as of 2026). But its graph is *organizational* — people, content, activity — not a code call-graph; not a head-to-head with a code-structure engine.
- **[Qodo](https://www.qodo.ai/blog/rag-for-large-scale-code-repos/)** has the strongest open code-embedding model and a serious multi-repo context engine — proof that great chunking and embeddings go far on semantic recall. The wedge: even the best chunker cannot return a resolved call edge or a verifiable blast radius.

---

## FAQ

**Is graph RAG better than vector RAG for code?**
Not strictly. Vector RAG gives strong recall on fuzzy natural-language queries but is approximate, structure-blind, and goes stale on edit. Graph RAG is deterministic and answers blast-radius and call-chain questions, but it needs upfront parsing and symbol resolution. The honest answer is hybrid: fuse BM25 lexical, vector semantic, and graph-centrality signals with Reciprocal Rank Fusion rather than picking one.

**Why do AI coding agents use grep instead of vector search?**
Determinism and freshness. Grep returns exact matches, needs no index, and reads the current file, so it never serves a stale embedding; it also fails loudly instead of returning a quietly wrong match. The catch is that grep is structure-blind — it cannot return resolved call or implements edges — which is why [a code knowledge graph](/gortex/code-knowledge-graph-for-llms/) plus vector retrieval beats grep alone.

**What is Reciprocal Rank Fusion (RRF) in code search?**
RRF merges ranked lists from different retrievers (BM25, vector, graph) by summing `1 / (k + rank)` for each document, with a default `k` of 60. Because it uses only rank positions and ignores raw scores, it sidesteps the fact that BM25 scores and cosine similarities live on different, incomparable scales. Hybrid retrieval fused with RRF consistently outranks either lexical or vector search alone.

**Knowledge graph vs vector database for code RAG — which should I use?**
Use both. A vector database excels at semantic "find code that does X" on fuzzy queries. A code knowledge graph gives verifiable edges (calls, implements, references) for structural questions like "what breaks if I change this?". Hybrid engines fuse lexical, vector, and graph signals so you do not have to choose — while pure-vector tools can still win on purely semantic natural-language recall at scale.

**Do code embeddings go stale when I edit my code?**
Yes. Every edit makes the affected chunk's embedding stale until it is re-embedded, and keeping a vector index current for an actively developed codebase adds real pipeline complexity. Engines that patch the graph incrementally — Gortex applies a roughly 200 ms per-file patch via file-watching — keep retrieval fresh without a full re-embed. This is part of why [a large codebase never fits the context window](/gortex/codebase-too-large-for-context-window/) cleanly with a static index.

---

## Bottom line

Stop reading "graph RAG vs vector RAG for code" as a fight. Vector RAG wins fuzzy semantic recall and loses on structure, boundaries, and freshness. Grep wins determinism and freshness and loses on structure. Graph RAG wins structure and determinism and costs you an upfront parse. The retrieval that holds up under a real, changing codebase fuses all three — lexical, semantic, graph-centrality — with Reciprocal Rank Fusion, tuned per query. If your questions are purely semantic and your corpus is huge, a pure-vector tool may serve you better; be honest about your query shape. But most agent questions about code are structural, and most code changes daily — which is where graph + vector + lexical, with deterministic edges and incremental patches, earns its place. Measure it on your own repo first. And if most of your agent's budget goes to finding code at all, start with [why your AI agent wastes most of its tokens on code retrieval](/gortex/ai-agent-token-waste-code-retrieval/).

[github.com/zzet/gortex](https://github.com/zzet/gortex)
