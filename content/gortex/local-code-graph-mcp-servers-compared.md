---
title: "Codanna alternative? Local code-graph MCP servers compared"
description: "A codanna alternative comparison: codanna, ChunkHound, Serena, Context7 and Gortex side by side — graph vs embeddings vs LSP vs docs, by use case."
date: 2026-05-28
lastmod: 2026-06-06
draft: false
slug: "local-code-graph-mcp-servers-compared"
keywords: ["codanna alternative", "chunkhound alternative no openai key", "serena mcp alternative", "context7 alternative for own codebase", "local code graph mcp comparison", "best code mcp server", "local code intelligence mcp server", "code graph mcp"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You wired an MCP server into your coding agent so it would stop grepping blind, and now you have a second problem: which one. Search "local code MCP server" and you get four serious projects — codanna, ChunkHound, Serena, Context7 — plus a dozen aggregator pages that list them without telling you they do fundamentally different things. So you install one, discover it can't do the thing you actually needed, and start over.

This is a fair, current **codanna alternative** comparison, written for someone who has to pick. The short version: these tools specialize on different axes. codanna is a fast read-only graph. ChunkHound is the strongest semantic search in this group. Serena edits code at the symbol level. Context7 doesn't index *your* code at all — it serves library docs. None of them is strictly better than the others, and one of them probably isn't even in the category you think it is.

{{< lead >}}There is no single best local code-graph MCP server — they use four different index paradigms. codanna and Gortex build a structural graph (calls, references, dependencies) for deterministic "who calls this / what breaks" answers. ChunkHound does semantic embeddings search for fuzzy "find code that does X." Serena drives a language server (LSP) for compiler-grade symbol editing and refactoring. Context7 fetches public library documentation and does not index your private repo at all. Pick by need: fast read-only graph → codanna; best semantic recall → ChunkHound; symbol-level refactoring → Serena; current library docs → Context7; graph + hybrid search + blast-radius + multi-repo + refactoring in one binary → Gortex.{{< /lead >}}

If the term "graph" here is new to you, read [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/) first — this article assumes you know the difference between a resolved edge and an embedding neighbor.

---

## First: Context7 is a different category (the "context7 alternative for own codebase" trap)

The most common confusion in this space deserves the top of the page. People try [Context7](https://github.com/upstash/context7) expecting it to index their repository, find that it can't, and go searching for a "Context7 alternative for my own codebase." That search is correct — but the premise is a category error worth understanding.

Context7 (by Upstash) supplies **up-to-date documentation for public third-party libraries**, injecting current, version-specific API docs and examples into your prompt so the model stops hallucinating against stale training data. It exposes two MCP tools — `resolve-library-id` and `get-library-docs` — and runs hosted, with near-zero setup, free at basic rate limits ([launch blog](https://upstash.com/blog/context7-mcp)). What it does *not* do is read your private code. There is no index of your repo, no call graph, no symbol search over your modules.

So the honest framing: Context7 is **complementary**, not a competitor, to a codebase indexer. Keep it for library docs; add a real code-intelligence server for your own code. The rest of this comparison is about that second category — tools that index *your* repository.

---

## The four index paradigms, plainly

Before the table, the mental model. These tools differ less in features than in what they build at index time. Get this right and the rest of the choice is easy.

| Paradigm | What it builds | Answers well | Examples |
|---|---|---|---|
| **Graph** | Resolved edges: calls, implements, references, dependencies | "Who calls this? What breaks if I change it?" — deterministic, verifiable | codanna, Gortex |
| **Embeddings / vector** | Vectors over code chunks, ranked by similarity | "Find code that does X" — fuzzy, natural-language recall | ChunkHound (also codanna, Gortex) |
| **LSP** | Live language-server symbol info, on demand | Compiler-grade symbol facts + editing | Serena |
| **Hosted docs** | Index of public library documentation | "Current API for library Y" | Context7 |

A graph stores facts the parser resolved — it knows that `handleAuth` *calls* `validateToken` because it traced the call, not because the two functions read similarly. Embeddings give you semantic recall a graph can't: a query like *"where do we rate-limit requests"* finds the right code even with no keyword match. LSP delegates to the same compiler your IDE uses, so its symbol facts are exact and its edits are safe. Hosted docs are a different problem entirely.

The interesting design question is whether a tool commits to one paradigm or fuses several. Most commit to one. We'll come back to that.

---

## Do you even need a codanna alternative? Codanna: fast, native, read-only graph + local semantic search

[codanna](https://github.com/bartolli/codanna) (Apache-2.0, written in Rust) positions itself as "X-ray vision for your agent": structural understanding — call graphs, implementations, dependencies — plus local semantic search, with an emphasis on speed (it advertises sub-10ms lookups and 75,000+ symbols/second indexing). The pitch is "context-first coding, no grep-and-hope loops," and "instant answers when an LSP is too slow."

Its semantic search runs locally with **no API key**: it uses a FastEmbed model (`all-MiniLM-L6-v2`, ~150 MB downloaded on first use), with an OpenAI-compatible embedding server as an *optional* upgrade ([codanna semantic search docs](https://docs.codanna.sh/features/semantic-search)). That makes it a genuine local graph-plus-semantic server you can run air-gapped.

Genuine strength: it's fast, native, permissively licensed, and combines structural and semantic retrieval in one tool. The honest limit: codanna is **read-only**. It's an intelligence and search server — it does not edit or refactor your code. (Its multi-repo/workspace story was unconfirmed as of writing; check the repo if you need it.)

---

## ChunkHound: the strongest semantic search — and it does NOT need an OpenAI key

[ChunkHound](https://github.com/chunkhound/chunkhound) (MIT, Python) bills itself as "local-first codebase intelligence." It is an embeddings-plus-regex search server, and its differentiator is genuinely interesting: **cAST**, AST-aware chunking based on [CMU research (arXiv:2506.15655)](https://arxiv.org/abs/2506.15655), which splits code along syntactic boundaries instead of fixed character windows. The paper reports +4.3 recall on RepoEval and +2.67 pass@1 on SWE-bench versus naive chunking. If your workflow is "research my codebase" — multi-hop semantic questions, architecture summaries, cited reports, even indexing docs and PDFs — ChunkHound's recall is the best in this group.

Now the myth, killed directly: **ChunkHound does not require an OpenAI API key.** As of 2026-06-06, its regex search needs no key at all, and its semantic search runs fully local via [Ollama](https://chunkhound.ai/) (e.g. `qwen3-embedding`) or vLLM. OpenAI and VoyageAI are *optional* providers. The project describes itself as "100% local," "code never leaves your machine," and "Free forever." If you saw an older claim that it's OpenAI-gated, that's stale.

Genuine strength: the strongest semantic chunking and recall in this group, fully local, permissive license. The honest limit: ChunkHound has **no structural graph** — it can't deterministically tell you who calls a function or what breaks if you change a signature — and it is **read-only**. It finds code; it doesn't model relationships or edit them.

---

## Serena: LSP-backed, the most mature symbol-level editing

[Serena](https://github.com/oraios/serena) (MIT, by oraios) calls itself "the IDE for your agent," and it has the largest mindshare here (~25k stars as of 2026-06-06). Instead of embeddings or a precomputed graph, it drives **language servers over LSP**, operating at the symbol level rather than on line numbers or string matches. That means compiler-grade accuracy across 40+ languages, with no vector index to build.

Its editing toolkit is the most complete in this comparison: `find_symbol`, `find_referencing_symbols`, `find_implementations`, `replace_symbol_body`, `insert_before_symbol` / `insert_after_symbol`, `safe_delete`, and `rename` (via LSP) ([Serena docs](https://oraios.github.io/serena/01-about/000_intro.html)). One caveat to state plainly: `move` and `inline` refactors are available **only through Serena's paid JetBrains plugin** (free trial) — the core MCP toolkit is free MIT, but don't conflate the two.

Genuine strength: the most mature symbolic editing and refactoring here, with the precision that comes from delegating to a real language server. The honest limit: Serena has no whole-repo vector index for fuzzy semantic recall, and no precomputed blast-radius — it answers structural questions live, per query, rather than from a baked graph you can sweep in bulk.

---

## A fair comparison table

One table, all five, honest rows. Star counts and tier facts are as of 2026-06-06 and move fast.

| | **codanna** | **ChunkHound** | **Serena** | **Context7** | **Gortex** |
|---|---|---|---|---|---|
| Language | Rust | Python | Python | TypeScript | Go |
| License | Apache-2.0 | MIT | MIT | MIT | Apache-2.0 |
| Index type | Graph + embeddings | Embeddings (cAST) | LSP (live) | Hosted library docs | Graph + hybrid (BM25+vector+graph) |
| Semantic search | Yes (FastEmbed) | Yes (cAST, strongest) | No | n/a | Yes (baked GloVe; opt-in MiniLM/OpenAI) |
| Local / no API key | Yes | Yes (Ollama/vLLM) | Yes | Hosted (optional key) | Yes (zero-setup default) |
| Multi-repo | Unconfirmed | Per-index | Per-project | n/a (libraries) | First-class + cross-repo contracts |
| Impact / blast-radius | Partial (graph) | No | Live, per-query | No | Precomputed, sub-millisecond |
| Refactor / edit | Read-only | Read-only | Yes (LSP; move/inline = paid plugin) | Read-only | Yes (graph-verified; move/inline Go-only) |
| Indexes *your* repo | Yes | Yes | Yes | **No** (library docs) | Yes |
| Stars (2026-06-06) | 687 | 1,299 | 25,001 | 56,858 | — |

The read-only vs editing split is the line most people miss: **codanna and ChunkHound only read; Serena and Gortex can edit** (Serena via LSP, Gortex graph-verified). And note the shared good news — codanna, ChunkHound (local), and Gortex all do **default-on semantic search with no API key required**. That's the modern baseline; treat any tool that *mandates* a cloud key as the exception.

---

## Where Gortex fits — and where it doesn't

Each tool above commits to one axis. The wedge for [Gortex](https://github.com/zzet/gortex) (Apache-2.0) is that it combines them: a deterministic knowledge graph *and* hybrid retrieval *and* structural editing *and* multi-repo, in one binary. Whether that union is worth it depends entirely on whether you need more than one axis — if you only need one, the specialist is often the cleaner choice.

What the union looks like concretely:

- **Graph + hybrid retrieval in one ranker.** Gortex isn't graph-instead-of-vectors. It fuses BM25, semantic vectors, and graph-centrality reranking via Reciprocal Rank Fusion (RRF), with a per-query alpha that tunes the lexical↔semantic mix. The graph gives deterministic, verifiable edges that embeddings only approximate. See [graph RAG vs vector RAG for code](/gortex/graph-rag-vs-vector-rag-for-code/) for the retrieval mechanics.
- **Precomputed blast-radius on every edit.** Where Serena answers "what references this?" live and ChunkHound can't answer it at all, Gortex precomputes a depth-3 reach index. Measured impact analysis p95 is 0.01ms — 100× under a 1ms budget — so asking "what breaks if I change this?" on every edit is cheap, not a special operation.
- **Graph-verified refactoring.** `rename_symbol` across files, `safe_delete_symbol` with an orphan-cascade gate, `verify_change` against all callers and interface implementors, and dependency-ordered `batch_edit`. Honest limit: `move_symbol` and `inline_symbol` are **Go-only for now** — Serena's LSP-driven editing covers more languages today.
- **First-class multi-repo with cross-repo API contracts.** Provider↔consumer matching for HTTP routes, gRPC, GraphQL, message topics, and env vars across repos — the kind of thing none of the single-repo tools above attempt.
- **257 languages** across three parsing tiers, and one install auto-configures **15 AI coding agents** it detects (Claude Code, Cursor, Windsurf, Cline, VS Code/Copilot, and more).

On token economy, the reason to query a graph instead of reading files: `get_symbol_source` returns ~80% fewer tokens than reading the whole file, and across a session the headline is **3–50× fewer tokens per response** (the 50× peak is on identifier lookups, not a flat average). For a self-hosted, no-key setup, see [local-first, open-source code intelligence — no cloud, no API key](/gortex/local-first-code-intelligence-no-cloud/).

Now the concessions, because they matter. Gortex is a binary plus a daemon, so it is **heavier to bootstrap** than a hosted tool like Context7 or a single-purpose script. It has **no browser/WASM mode**. And its *default* embeddings are lightweight GloVe-50d (baked in, zero-setup) — recall on concept and multi-hop semantic queries is modest compared to a dedicated embeddings tool like ChunkHound with a strong local model. Gortex dominates on *exact-tier* symbol queries and structural questions; for pure semantic research over a large codebase, ChunkHound's cAST is the better instrument. Pick the union only if you actually need the union.

If you're choosing an MCP server primarily to give an agent codebase context, the broader walkthrough is [the code-intelligence MCP server your setup is missing](/gortex/mcp-server-for-codebase-context/); if you're replacing a hosted indexer, see [the self-hosted Sourcegraph and Cody alternative for AI agents](/gortex/sourcegraph-cody-alternative-self-hosted/).

---

## Recommendation by use case

No blanket winner. Match the tool to the job:

| You want… | Use | Why |
|---|---|---|
| A fast, read-only local graph + semantic search | **codanna** | Native Rust speed, Apache-2.0, FastEmbed local embeddings, no key |
| The best semantic recall / "research my codebase" | **ChunkHound** | cAST AST-aware chunking, fully local, indexes docs + PDFs |
| Compiler-grade symbol editing and refactoring | **Serena** | LSP accuracy, 40+ languages, mature editing toolkit |
| Current docs for public libraries | **Context7** | Up-to-date version-specific library docs in the prompt |
| Graph + hybrid + blast-radius + multi-repo + refactor in one | **Gortex** | The union of the above, if you need more than one axis |

And nothing stops you from running two — Context7 for library docs alongside any one of the codebase indexers is a common, sensible pairing.

---

## FAQ

### What is the best codanna alternative for a local code-graph MCP server?

It depends what you use codanna for. For the same Rust-speed, read-only local graph plus semantic search, codanna itself is solid (Apache-2.0, FastEmbed local embeddings, no API key). If you also want graph-verified editing/refactoring, sub-millisecond blast-radius, multi-repo with cross-repo API contracts, and 257 languages in one binary, Gortex (Apache-2.0) covers that union. For pure semantic recall, ChunkHound; for compiler-grade symbol editing, Serena.

### Does ChunkHound require an OpenAI API key?

No. As of 2026-06-06, ChunkHound's regex search needs no key, and its semantic search runs fully local via Ollama or vLLM. OpenAI and VoyageAI are optional providers. The project bills itself as "100% local" and "Free forever" (MIT), so you can run it with zero cloud calls.

### Can Context7 index my own codebase?

No. Context7 (by Upstash) indexes up-to-date documentation for public third-party libraries and injects it into your prompt so the model uses current APIs instead of stale training data. It does not index your private repository. To give an agent structural understanding of your own code, use a codebase indexer — codanna, ChunkHound, Serena, or Gortex — and keep Context7 alongside it for library docs.

### What's the difference between a graph, embeddings, LSP, and docs MCP server?

A graph server (codanna, Gortex) models real relationships — calls, implementations, references, dependencies — for deterministic "who calls this / what breaks" answers. An embeddings server (ChunkHound; also part of codanna and Gortex) does semantic similarity search over code chunks, good for fuzzy "find code that does X." An LSP server (Serena) drives language servers for compiler-grade symbol info and editing. A docs server (Context7) fetches public library documentation. Gortex is hybrid: it fuses BM25 + embeddings + graph-structure reranking.

### Which local code MCP server can actually refactor code, not just search it?

Serena and Gortex can edit; codanna and ChunkHound are read-only. Serena uses LSP for symbol-level edits (rename, replace_symbol_body, insert before/after, safe_delete; move/inline require its paid JetBrains plugin). Gortex offers graph-verified rename across files, safe_delete with orphan-cascade gating, and dependency-ordered batch edits (move/inline are Go-only for now).

---

## Bottom line

These four tools are not really competing — they occupy four different categories. Context7 serves library docs, not your repo; codanna gives a fast read-only graph; ChunkHound gives the strongest semantic recall; Serena gives compiler-grade symbol editing. Pick by the question you actually need answered. If that question is more than one of the above at once — graph navigation *and* hybrid search *and* blast-radius *and* multi-repo *and* refactoring — that union is what Gortex is built for, with the honest caveats that it's heavier to bootstrap, has no browser mode, and ships modest default embeddings you upgrade if you need them.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
