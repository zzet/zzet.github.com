---
title: "How does Cursor index your codebase? Embeddings vs a graph"
description: "How does Cursor index your codebase: the real chunk-embed-Merkle pipeline, where it misses, and how a structural graph resolves edges over similarity."
date: 2026-06-02
lastmod: 2026-06-06
draft: false
slug: "how-cursor-indexes-codebase-embeddings-vs-graph"
keywords: ["how does Cursor index your codebase", "cursor only reads first 250 lines", "incremental code indexing on file save", "repo map for LLM", "PageRank repo map symbol graph", "embeddings vs structural graph", "Merkle tree code indexing", "cursor @codebase misses"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You type `@codebase` and ask Cursor where a request gets authenticated. Most of the time it lands. Sometimes it returns a plausible function that's three layers from the real one, misses the middleware, or hands you half a function because the other half lived in a different chunk. You can't see why, because the index is invisible. So the question is fair and worth answering precisely: **how does Cursor index your codebase**, and why does that mechanism occasionally miss?

This post is the honest mechanics first — the chunk-embed-Merkle pipeline, what leaves your machine, and the genuine `cursor only reads first 250 lines` confusion — then a fair comparison of three retrieval philosophies: embeddings (Cursor), a ranked repo map (Aider), and a resolved structural graph. Gortex is in the last camp, and it appears late, with concessions.

> **Short answer:** Cursor splits your files into syntactic chunks, sends those chunks to its servers to be turned into vector embeddings, and stores the vectors plus metadata (line ranges, obfuscated file paths) in a remote vector database (turbopuffer). A Merkle tree of file hashes tracks changes, so editing a file re-embeds only the chunks whose hashes changed. Your raw source is held in memory during indexing and then discarded, not persisted. The limit: embeddings store *similarity*, not *relationships* — the index has no resolved edges, so it can't tell you what calls a function, what implements an interface, or what breaks if you change a signature.

---

## How does Cursor index your codebase, step by step

Lead with the mechanism, because it is well-engineered and most "Cursor is dumb" complaints come from misunderstanding it. The pipeline, per [Cursor's writeup on securely indexing large codebases](https://cursor.com/blog/secure-codebase-indexing) and the [codebase indexing docs](https://cursor.com/docs/context/codebase-indexing):

1. **Chunk locally.** Cursor splits each file into syntactic chunks — functions, classes, logical blocks — on your machine.
2. **Embed server-side.** Those chunks are sent to Cursor's servers and turned into vector embeddings with Cursor's own embedding model. (Older writeups say OpenAI; as of 2026 [turbopuffer's case study](https://turbopuffer.com/customers/cursor) attributes it to Cursor's own model — so treat the provider as "a code-tuned model," not a fixed vendor.)
3. **Store vectors, not source.** The vectors plus metadata — line ranges and *obfuscated* file paths — go into a remote vector database (turbopuffer). The raw source is held in memory during indexing and then discarded, not persisted in plaintext.
4. **Track changes with a Merkle tree.** Leaves are file hashes, parents hash their children. On a change Cursor walks only the branches where hashes differ and re-syncs those entries, so only the changed chunks are re-embedded.
5. **Query.** Your query is embedded, turbopuffer does nearest-neighbor search, returns obfuscated path + line range, and the client reads those chunks from local disk.

Operationally: indexing starts when you open a workspace, semantic search becomes usable at **80% completion**, and the index auto-syncs roughly **every 5 minutes**, processing only changed files. At turbopuffer's scale this is real engineering — the case study cites **1T+ vectors across 80M+ namespaces** (one per codebase instance). To be clear: this is a *good* design for fuzzy, zero-config semantic search. The criticism below is not "Cursor is bad" — it's "embeddings approximate structure, and that approximation has predictable failure modes."

### Set the "Cursor only reads first 250 lines" record straight

The `cursor only reads first 250 lines` claim is real but is about a *different system* than the index — the agent's `read_file` tool, not the embedding index:

| System | What it does | The limit |
|---|---|---|
| **Embedding index** | Chunks + vectors for `@codebase` semantic search | No documented per-file line cap |
| **`read_file` tool** | Agent reads a file on demand during a task | ~250 lines standard mode (sometimes +250), up to **750 lines in Max mode** |

Reading a whole file is only allowed if you edited it or manually attached it; otherwise `should_read_entire_file=true` still returns just the first lines ([Cursor forum thread](https://forum.cursor.com/t/read-files-in-chunks-of-up-to-250-lines/52597), confirmed as of 2026). So if your agent "missed" something past line 250, that's the on-demand read tool being frugal about context cost — not the search index truncating your repo. Two separate systems.

---

## Why chunk-embedding indexing misses

The educational core: embedding indexes have five structural limits, independent of how good the model is.

**No resolved edges.** An embedding is a point in vector space; "near" means "semantically similar." It does not encode that function `A` *calls* `B`, that class `C` *implements* interface `I`, or that field `f` is *written* here and *read* there. Ask "what calls `chargeCard`?" and a vector index returns things that *look like* `chargeCard`, not its actual callers. There is nothing to traverse.

**No blast radius.** With no edges, the index can't answer "what breaks if I change this signature?" — that requires walking from a definition to every resolved reference and interface implementor, exactly the traversal embeddings don't support. This is the gap explored in [graph RAG vs vector RAG for code](/gortex/graph-rag-vs-vector-rag-for-code/).

**Chunk-boundary breakage.** Fixed or syntactic chunking can split a function across two chunks. Embedding models trained largely on natural language don't inherently know a method's enclosing class or its dependencies, so related code fragments — even *perfect* retrieval returns half a function ([AST-aware chunking](https://supermemory.ai/blog/building-code-chunk-ast-aware-code-chunking/); [chunking strategies for RAG](https://weaviate.io/blog/chunking-strategies-for-rag)). AST-aware chunking helps, but the unit is still a chunk, not a symbol.

**Server dependency.** Embeddings, the vector DB, and nearest-neighbor search all live on Cursor's infrastructure. No connection, no semantic search — a hard stop for air-gapped or strict-compliance shops.

**Embedding-inversion residual risk.** Cursor itself concedes that even though raw source isn't persisted, a breached vector DB could have its embeddings partially inverted back toward content ([Simon Willison's security notes](https://simonwillison.net/2025/May/11/cursor-security/)). A residual risk, not a routine one — but one the vendor names, not a critic's invention.

None of this makes embeddings useless. It makes them *similarity search*, which is a different thing from *structure*.

---

## The honest menu: embeddings vs repo map vs structural graph

There's more than one non-Cursor way to give an agent codebase context, and two of them are genuinely smart.

| Approach | How it builds context | Edges? | Local? | Strength | Limit |
|---|---|---|---|---|---|
| **Cursor (embeddings)** | Chunk → embed server-side → vector DB | No | No (cloud) | Invisible, zero-config, scales | Approximates structure; server dep |
| **Aider (repo map)** | tree-sitter symbols → PageRank rank → token budget | Reference graph, ranked | Yes | No API key for the map; deterministic ranking | Ranked map, not a live queryable graph |
| **Windsurf** | Local full-repo index + M-Query + Riptide rerank | No (rerank, not edges) | Index local | Reranking quality, indexes unopened files | Still similarity-based retrieval |
| **Repomix** | Serialize whole repo to one file | No | Yes | Dead-simple, deterministic | One-shot dump; re-bills every turn |
| **Structural graph (Gortex)** | Parse → resolve edges → in-memory graph + hybrid search | Yes (resolved) | Yes | Real calls/implements/references; "what breaks?" | Binary + daemon; no in-browser story |

### Aider's PageRank repo map — the smart non-embedding contrast

[Aider](https://aider.chat/2023/10/22/repomap.html) uses *no embeddings at all* — the most instructive comparison here. It parses your repo with tree-sitter to find every symbol definition and reference, builds a graph where files are nodes and references are edges, ranks symbols with a personalized PageRank, and packs the most-referenced ones into a token budget (`--map-tokens`, default ~1k, confirmed in the [Aider docs](https://aider.chat/docs/repomap.html)).

The ranking weights references, biasing toward the files in your current chat and symbols you've mentioned. (Precise edge-weight multipliers circulating online — ~10× for mentioned symbols, ~50× for chat files — come from [DeepWiki](https://deepwiki.com/Aider-AI/aider/4.1-repository-mapping-system) and community reimplementations, not the original blog, so read them directionally: chat and mentioned files weigh more.)

This is a `PageRank repo map symbol graph` done right: local, no server, no API key for the map, ranking by structural importance — reference frequency — rather than fuzzy similarity. Its honest limit: it produces a *ranked context map* sized to a token budget, not a *live, queryable graph* you can ask "who calls this?" or "what breaks?" against. A great `repo map for LLM` input; not an impact-analysis engine.

**Windsurf** ([context docs](https://docs.windsurf.com/context-awareness/overview)) indexes the whole local codebase, including unopened files, and layers an "M-Query" step plus the Riptide trained reranker over cosine similarity; treat its cited recall gains as a vendor claim (unconfirmed) and trained reranking as the real point. **Repomix** ([GitHub](https://github.com/yamadashy/repomix)) isn't an index at all — it packs the whole repo into one AI-friendly file with token counts, ideal when the repo fits the context window, the wall described in [your codebase will never fit the context window](/gortex/codebase-too-large-for-context-window/). The deeper trade-off of concatenation tools is in [beyond Repomix and Gitingest — query a graph, don't concatenate](/gortex/repomix-gitingest-alternative-graph/).

---

## The structural-graph alternative: resolved edges, on your machine

A structural-graph engine indexes your repo into a [code knowledge graph for LLMs](/gortex/code-knowledge-graph-for-llms/) — an in-memory graph of *resolved* edges (calls, implements, references) — instead of cloud vectors. The difference from embeddings is similarity versus structure: a graph edge is a verifiable fact ("`A` calls `B`"), not a distance.

[Gortex](https://github.com/zzet/gortex) is in this camp, and the framing matters: it is **graph + vector**, not "graph instead of vectors." Retrieval is hybrid — BM25/FTS5 lexical, a default-on baked GloVe-50d semantic channel (no API key; opt-in MiniLM/Ollama/OpenAI for stronger recall), fused with Reciprocal Rank Fusion, then **graph-centrality reranking** that floats structurally-central results up. The graph is the differentiator on top of similarity, not a replacement for it. Three properties address the embedding limits above:

**Incremental on-save patches — the spirit of Cursor's Merkle sync.** Cursor re-embeds changed chunks; Gortex *patches the graph*. An `fsnotify` watch fires on save, debounced ~150ms, applying a surgical per-file graph patch — ~200ms incremental re-index versus a 3–5s full rebuild. This is `incremental code indexing on file save`, but the unit updated is resolved edges, not vectors, and it never leaves your machine.

**Whole-symbol reads — no chunk-boundary breakage.** Because the unit is a symbol, not a chunk, `get_symbol_source` returns a complete function or class — never half of one split at an arbitrary boundary — at roughly **80% fewer tokens** than reading the whole file. Per response, Gortex reports **3–50× fewer tokens** versus naive file reads (the ~50× peak is on identifier lookups, not a flat average).

**"What breaks?" as a first-class query.** This is the answer embeddings structurally cannot give. `get_callers` / `get_dependents` traverse resolved edges, and `explain_change_impact` / `verify_change` return a risk-tiered blast radius — checked against a precomputed depth-3 reach index at sub-millisecond lookups (measured impact p95 ≈ 0.01ms).

```bash
# Cursor: similarity — "what looks like chargeCard?"
@codebase where is chargeCard

# Graph: structure — verifiable edges
gortex query callers chargeCard   # every resolved caller, no false positives
```

For the blast radius before you edit, the agent calls the `explain_change_impact` MCP tool on `chargeCard`, which returns a risk-tiered list of everything the change reaches.

Gortex is Apache 2.0, runs fully local with no model download to start, exposes the graph over CLI + MCP + HTTP, and supports 257 languages with 100+ MCP tools. For wire economy it can emit GCX1, measured at a median **−27.4% tokens** versus JSON at the same fidelity.

### Honest concessions

Cursor wins on the thing most people actually weigh: it is **invisible and zero-config**. No binary, no daemon, no model download, auto-sync, and a shareable team index — many developers rightly prefer that friction-free UX, and Gortex is a heavier binary-plus-daemon with no in-browser story. Aider's repo map needs no setup at all. And Gortex's own limits are real: default GloVe-50d embeddings are lightweight (opt into MiniLM/OpenAI for best recall); semantic recall on concept/multi-hop tiers is modest (R@5 ≈ 25–30%) — its dominance is on *exact* symbol queries (BM25 R@5 ≈ 96.8%); and `move_symbol` / `inline_symbol` are Go-only for now. Benchmarks are single-machine; absolute timings vary 2–5× by hardware.

The point isn't that one wins outright. It's that embeddings and a graph answer *different questions* — and for "what calls this / what breaks / read the exact symbol," resolved edges beat similarity.

---

## FAQ

### How does Cursor index your codebase?

Cursor splits your files into syntactic chunks, sends those chunks to its servers to be turned into vector embeddings, and stores the vectors plus metadata (line ranges and obfuscated file paths) in a remote vector database (turbopuffer). A Merkle tree of file hashes tracks changes, so editing a file re-embeds only the chunks whose hashes changed. Indexing starts when you open a workspace, semantic search becomes usable at 80% completion, and the index auto-syncs about every 5 minutes. Your raw source is held in memory during indexing and then discarded — not persisted in plaintext on the server.

### Does Cursor only read the first 250 lines of a file?

That limit is about Cursor's `read_file` agent tool, not its search index. In standard mode the tool returns roughly the first 250 lines (sometimes +250 more); Max mode raises it to 750. Reading a whole file is only allowed if you edited it or manually attached it — otherwise even `should_read_entire_file=true` returns just the first lines. The embedding index has no documented 250-line cap; the cap is a context/cost tradeoff in how the agent reads files on demand.

### What are the limits of embedding-based code indexing like Cursor's?

Embeddings store similarity, not relationships, so the index has no resolved edges — it can't tell you what calls a function, what implements an interface, or what breaks if you change a signature, because there's nothing to traverse. Chunking can split a function across boundaries and fragment context, and the pipeline depends on Cursor's servers and a cloud vector DB (Cursor notes embedding inversion as a residual risk if that DB were breached). It remains excellent zero-config fuzzy search; it just approximates structure rather than resolving it.

### How is Aider's PageRank repo map different from Cursor's embeddings?

Aider uses no embeddings. It parses your repo with tree-sitter to find every symbol definition and reference, builds a graph where files are nodes and references are edges, ranks symbols with personalized PageRank, and packs the most-referenced ones into a token budget (default ~1k tokens). It's local, needs no server or API key for the map, and ranks by structural importance rather than fuzzy similarity — a smarter, deterministic non-embedding approach. Its limit: it produces a ranked context map, not a live queryable graph with resolved call/implements edges you can ask "what breaks?" against.

### What's a structural-graph alternative to embedding indexing?

A structural-graph engine like Gortex indexes your repo into a local, in-memory graph of resolved edges — real calls, implements, references — instead of cloud vectors. It updates incrementally on file save (~150ms debounce, ~200ms incremental re-index instead of a full rebuild), reads whole symbols rather than arbitrary line windows so it never splits a function, and answers "what calls this?" and "what breaks if I change it?" with sub-millisecond impact analysis. Retrieval is hybrid — BM25 + lightweight built-in embeddings + graph-centrality reranking — so it's graph *plus* vectors with deterministic edges, all on your machine.

---

## Bottom line

Cursor's index is well-built and frictionless: chunk locally, embed server-side, store vectors keyed by a Merkle tree, re-embed only what changed. That gives you fast, invisible, zero-config semantic search, and many developers correctly prefer it. What it cannot give you is structure: with no resolved edges it approximates "what calls this" with "what looks like this," can split a function at a chunk boundary, and depends on a cloud vector DB. Aider's repo map shows you can rank structure locally without embeddings; a structural graph goes further and makes the edges queryable. If your hardest questions are "who calls this, what implements this, what breaks if I change it," you want resolved edges, not similarity — graph plus vectors, on your machine.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
