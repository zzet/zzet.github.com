---
title: "The MCP server for codebase context you're missing"
description: "Pick an MCP server for codebase context: a fair rubric scoring claude-context, Serena, codanna, and Context7, and why a graph-native server wins."
date: 2026-06-04
lastmod: 2026-06-06
draft: false
slug: "mcp-server-for-codebase-context"
keywords: ["MCP server for codebase context", "best MCP server for coding agents", "semantic code search MCP server", "code search MCP for Claude Code", "code intelligence mcp", "mcp server for codebase"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

Your agent is blind to your repo. You ask it to change a function, and it runs `grep`, opens five files, re-reads two of them, opens five more, and still hands you an edit that misses a caller. The model is capable; it just can't *see* the codebase. The integration point that's supposed to fix this is MCP — but which **MCP server for codebase context** actually gives an agent a working picture of your code, and which just bolt on one more search box?

That confusion is the expensive part. "MCP server" now spans library-docs lookups, vector search, LSP bridges, and graph engines — tools that solve genuinely different problems but all advertise "context for your agent." Pick the wrong one and your agent is still grepping; pick another and you've added an API key and a vector database for search-only retrieval.

So this article gives you the educational core — what MCP actually is (a socket, not a brain) and the eight things a codebase-context server *should* do — then a fair, scored comparison of the real options before landing on where a graph-native server fits. If you never install anything from here, you'll still leave able to evaluate one yourself.

> **Short answer:** An MCP server for codebase context is a program that connects to your coding agent over JSON-RPC and exposes capabilities through three primitives — Tools, Resources, and Prompts. A *good* one lets the agent search symbols, navigate, read one function instead of a whole file, find callers and usages, and analyze the impact of a change, so it stops grepping and re-reading blindly. MCP is the socket; the server is what gives the agent real understanding of your repo. Most servers ship only Tools — and most ship only search.

## What MCP actually is (and what it isn't)

The Model Context Protocol is an open standard — [introduced and open-sourced by Anthropic](https://www.anthropic.com/news/model-context-protocol) — for connecting LLM applications to the systems where data and tools live. It's LSP-inspired and rides on [JSON-RPC 2.0 messages](https://modelcontextprotocol.io/specification/2025-11-25). Since then it's spread well past one vendor: OpenAI adopted it in 2025, and [governance moved to a Linux Foundation directed fund in late 2025](https://en.wikipedia.org/wiki/Model_Context_Protocol). It is, by now, the default way agents talk to external capabilities.

The thing to internalize: MCP is the *socket*, not the intelligence. The spec defines exactly three server-side primitives:

| Primitive | What it is (per the spec) | Who drives it |
|---|---|---|
| **Tools** | "Functions for the AI model to execute" | model-controlled |
| **Resources** | "Context and data, for the user or the AI model to use" | application-controlled |
| **Prompts** | "Templated messages and workflows for users" | user-controlled |

Almost every MCP server you'll meet ships only **Tools** — Resources and Prompts are rare. That matters because the protocol is just plumbing; the *brain* is whatever plugs into the socket. Two servers can both speak MCP perfectly and give your agent wildly different amounts of understanding. The real question is never "does it support MCP?" It's "what's on the other end?"

## Why your agent is blind without one

Left to itself, a coding agent reaches for the tools it always had: shell, grep, file reads. Claude Code [spawns Explore agents that scan files with grep, glob, and Read](https://dev.to/badmonster0/your-ai-coding-agent-is-blind-heres-the-fix-569n); without a dependency graph, those agents "load 10–30 files when only 3–5 are needed." Cursor's index is more sophisticated — a [Merkle tree of file hashes, tree-sitter chunking, embeddings in Turbopuffer, re-synced roughly every five minutes](https://cursor.com/blog/secure-codebase-indexing) — but it's still chunk retrieval, not navigation.

The failure mode is the same across agents:

- **Grep-and-read loops.** Match text, open the file, realize it's the wrong one, repeat.
- **Re-reading.** The same file gets pulled into context two or three times a session.
- **Over-fetching.** Whole files loaded for one function; 25 files loaded for a 5-file change.
- **Missed edges.** A caller in another module the agent never opened, so the edit silently breaks it.

Each of those is wasted tokens and a wrong-answer risk. The ecosystem is converging on MCP as the place codebase awareness belongs — [Continue.dev's `@Codebase` provider is deprecated as of 2026](https://docs.continue.dev/reference/deprecated-codebase), still selectable as legacy but with users steered to Agent-mode tools and MCP servers. The fix isn't a faster grep; it's a server that *resolves* code into structure.

## What an MCP server for codebase context should do: an 8-point rubric

Here's the rubric. Strip the marketing and a codebase-context MCP server should do these eight things:

1. **Symbol search** — find a symbol by name or by natural-language intent ("the auth flow"), not just by text.
2. **Navigation** — jump to declarations, implementations, type hierarchy.
3. **Read a symbol, not a file** — return one function's source, not 400 lines around it.
4. **Callers / usages** — "who calls this?" and "where is this referenced?" precisely.
5. **Impact analysis** — "what breaks if I change this signature?"
6. **Multi-repo** — query across repositories, including cross-repo contracts.
7. **Freshness** — stay current as files change, without a manual reindex every time.
8. **One-command install** — configure your agents without hand-editing JSON for each.

No tool has to score 8/8 to be useful. But the rubric lets you see *what you're buying* — and where the gaps are.

## The honest solution menu

These are the real options, treated fairly. Star counts, versions, and **licenses** are all snapshots as of mid-2026 and can change — re-verify against each project's repository before you rely on them.

### claude-context (zilliztech) — vector search, search-only

[claude-context](https://github.com/zilliztech/claude-context) bills itself as "code search MCP for Claude Code — make your entire codebase the context for any coding agent," and it delivers on that narrow promise. It runs genuine hybrid retrieval — BM25 plus dense vectors — with AST-aware chunking and Merkle-tree incremental reindex, scaling to million-line repos. MIT, ~11.7k stars.

The caveats are structural. It exposes exactly four tools — `index_codebase`, `search_code`, `clear_index`, `get_indexing_status` — so it's search-only: no callers, usages, impact, navigation, or read-a-symbol. And to start it you need *both* an embedding provider (OpenAI, VoyageAI, Gemini, or Ollama — an API key unless you self-host) *and* a vector database (Milvus or Zilliz Cloud). High on symbol search, near-zero on the rest of the rubric.

### Serena (oraios) — LSP-native, elegant

[Serena](https://github.com/oraios/serena) is the one to respect. It's LSP-backed — operating "at the symbol level rather than line numbers or primitive patterns" across 40+ languages, with **no embeddings, no vector DB, no API key**. ~25k+ stars, MIT. Its tools are rich: find symbol, find referencing symbols, type hierarchy, find declaration/implementations, query external projects, plus *symbolic editing* — rename, move, inline, replace-body. Same engine your editor uses, and genuinely elegant.

Concede it openly: for symbol navigation and structured editing with no cloud dependency, Serena is excellent. The trade-offs are inherent to LSP — a startup/indexing cost, one language server per language (heavy in polyglot repos), and **exact-symbol only**, with no natural-language "find the auth flow" search. It also has no graph-wide impact analysis or cross-repo contracts, and freshness is tied to the language server, not a purpose-built daemon.

### codanna (bartolli) — lightweight local graph

[codanna](https://github.com/bartolli/codanna) is closest in spirit to a graph-native engine: local-first, tree-sitter parsing, with call graphs, implementations, and type relationships, plus 384-dim doc-comment embeddings (AllMiniLM-L6-v2, local, **no API key**) and Tantivy full-text search. Its tool set — `semantic_search_docs`, `find_symbol`, `find_callers` (configurable depth), `get_calls`, `analyze_impact` — already covers more of the rubric than search-only tools. Apache 2.0, ~687 stars, v0.9.x (~May 2026).

Caveats: the embeddings are doc-comment-only, so semantic recall is weak on undocumented code; ~14–15 languages; no documented multi-repo or cross-repo contracts; freshness via manual reindex (no live file-watch daemon documented); sub-1.0 maturity; and Tools only — no Resources or Prompts.

### Context7 (upstash) — a different problem (concede)

[Context7](https://github.com/upstash/context7) is the one *not* to confuse with the rest. ~56.9k stars, MIT, two tools (`resolve-library-id`, `get-library-docs`). It injects up-to-date, version-specific **library documentation** to stop API hallucination — "documentation straight from the source." It indexes *public libraries*, not your code, and doesn't search, navigate, or analyze your repo.

That's not a knock — it's a category clarification. Context7 solves library-docs hallucination, which is real and worth solving, and it's **complementary** to a codebase-context server, not a competitor. Run both: Context7 for "is this the current React Query API?" and a code-intelligence server for "where does *our* auth flow live and what calls it?"

### The scorecard

| Rubric | claude-context | Serena | codanna | Context7 |
|---|---|---|---|---|
| Symbol search | vector + BM25 | exact (LSP) | FTS + doc-embeds | n/a (libs) |
| Navigation | no | yes (LSP) | partial | no |
| Read-a-symbol | no | yes | yes | no |
| Callers / usages | no | yes | yes | no |
| Impact analysis | no | no | `analyze_impact` | no |
| Multi-repo | no | external projects | no | n/a |
| Freshness | Merkle reindex | LSP | manual reindex | live (libs) |
| NL semantic search | yes | no | doc-only | n/a |
| API key / vector DB needed | **yes (both)** | no | no | optional |
| License (as of 2026) | MIT | MIT | Apache 2.0 | MIT |
| Stars (mid-2026) | ~11.7k | ~25k+ | ~687 | ~56.9k |
| MCP primitives | Tools | Tools | Tools | Tools |

Read the table honestly and two gaps appear. The search-only tools miss navigation and impact entirely. The LSP tool misses natural-language search. None of the four does multi-repo contracts, and only one ships anything beyond Tools — none ship Resources *and* Prompts. That's the gap a graph-native server is built to fill.

## Why graph + vector beats vector-only and LSP-only

The two strong approaches each leave half the problem on the table.

**Vector-only** (claude-context, Cursor's index) finds chunks that are *semantically similar* — which only approximates relationships and routinely misses exact symbol matches, precisely [why coding agents still default to grep, not vectors](https://www.mindstudio.ai/blog/is-rag-dead-what-ai-agents-use-instead). It can't answer "who calls this?" because similarity isn't a call edge. **LSP-only** (Serena) gives deterministic, compiler-grade edges — but only for exact symbols, one language server at a time, with no natural-language search and no graph-wide impact across repos.

A graph-native server stores **deterministic, verifiable edges** — calls, implements, references — so it answers "who calls this?" and "what breaks if I change this signature?" *precisely*, not probabilistically. The strong version is *hybrid*: it fuses BM25 lexical, semantic embeddings, and graph-centrality reranking. It's **graph + vector, not graph vs vector** — the combination neither pure-vector nor pure-LSP gives you alone. This is the case for [a code knowledge graph your AI agent can actually navigate](/gortex/code-knowledge-graph-for-llms/), and the foundation of [local-first, open-source code intelligence with no cloud and no API key](/gortex/local-first-code-intelligence-no-cloud/).

## Gortex against the full rubric

[Gortex](https://github.com/zzet/gortex) is a graph-native code-intelligence engine that indexes a repo into an in-memory knowledge graph and exposes it over CLI, an MCP server, and HTTP. It's the one entrant designed to satisfy the whole rubric in one install — and notably, **the only one that uses all three MCP primitives**: 100+ tools, 16 resources, and 3 prompts.

Mapping it back to the eight points:

| Rubric point | Gortex |
|---|---|
| Symbol search | `search_symbols` (BM25, camelCase-aware) + default-on semantic, fused via RRF |
| Navigation | `find_declaration`, `find_implementations`, `get_repo_outline` |
| Read-a-symbol | `get_symbol_source` — ~80% fewer tokens, returns `tokens_saved` |
| Callers / usages | `get_callers`, `find_usages` — zero false positives, typed context |
| Impact analysis | `explain_change_impact` — sub-millisecond, p95 0.01ms |
| Multi-repo | `track_repository`, `query_project`, plus `contracts` for cross-repo APIs |
| Freshness | fsnotify daemon → surgical per-file graph patches, ~200ms incremental |
| One-command install | one curl, configures the 15 agents it detects |

A few specifics. Retrieval is **hybrid**, not "graph instead of vectors": BM25 plus semantic plus graph-centrality reranking, blended with Reciprocal Rank Fusion. Semantic search is **default-on with no API key** — it ships baked GloVe-50d embeddings (3.8MB, zero dependencies), with optional MiniLM/Ollama/OpenAI upgrades for stronger recall. Reading a symbol instead of a file saves **3–50× tokens per response** (peak ~50× on identifier lookups). Impact analysis runs against a precomputed reach index, so asking "what breaks?" on every edit is cheap. It covers **257 languages**, is **Apache 2.0**, and has **zero external dependencies to start** — no database, no model download, no network. For bolting that onto an editor that's already close, see [when @codebase isn't enough — adding a graph to Cursor and Cline](/gortex/when-codebase-context-isnt-enough-cursor-cline/); for the broader field, [local code-graph MCP servers compared: codanna, ChunkHound, Serena, Context7](/gortex/local-code-graph-mcp-servers-compared/).

```bash
# one install; configures every agent it detects (Claude Code, Cursor, Windsurf, …)
curl -fsSL https://get.gortex.dev | sh
```

### Honest limits

Gortex isn't free of trade-offs. It's a **binary plus a daemon** — heavier to bootstrap than a browser or WASM tool, with no in-browser story. `move_symbol` and `inline_symbol` are **Go-only for now**. The default GloVe embeddings are lightweight (great for zero-setup, no key), but for the strongest semantic recall you'll opt into MiniLM or OpenAI — and even then, recall on concept/multi-hop queries is modest; exact-symbol search is where it dominates. Benchmark numbers are single-machine figures that vary by hardware (reproducible via `gortex bench`). And as conceded above, Context7 still wins its own category — library docs — which Gortex doesn't try to cover.

## FAQ

### What is an MCP server for codebase context?

It's a program that connects to your AI coding agent over JSON-RPC and exposes capabilities through three MCP primitives — Tools (functions the model calls), Resources (data it reads), and Prompts (workflow templates). A codebase-context server uses those to let the agent search symbols, navigate, read one function instead of a whole file, find callers and usages, and analyze a change's impact — so it stops grepping and re-reading files blindly. MCP is the socket; the server is what gives the agent real understanding of your repo.

### What's the best MCP server for coding agents?

It depends on what you need. claude-context does vector + BM25 search but needs an embedding key and a vector database and offers search only. Serena is elegantly LSP-native with great symbol navigation and editing and no API key, but has no natural-language semantic search and a per-language server cost. codanna is a lightweight local graph with callers and `analyze_impact`. Context7 is for library docs, not your code. For the full rubric — search, navigation, read-a-symbol, callers/usages, sub-millisecond impact, multi-repo, and live freshness in one install — Gortex is the most complete: 100+ tools, 16 resources, 3 prompts, hybrid graph + semantic with no API key, Apache 2.0.

### Is Context7 a codebase-context MCP server?

No, and it's worth not confusing the two. Context7 injects up-to-date, version-specific documentation for *public libraries* to prevent API hallucination. It does not index, search, or navigate your own codebase. It's genuinely useful and complementary — but for understanding your repo you need a code-intelligence MCP server like Gortex, Serena, codanna, or claude-context.

### Do I need a vector database or API key for codebase context?

Not necessarily. claude-context requires both an embedding provider and a vector database. But Serena uses LSP with no key, codanna runs a local embedding model with no key, and Gortex ships default-on semantic search with baked GloVe-50d vectors (3.8MB, zero dependencies, no key) fused with BM25 and graph structure — with optional MiniLM/OpenAI upgrades for stronger recall.

### How is a graph MCP server different from semantic code search?

Pure vector search finds chunks that are semantically similar but only approximates relationships and often misses exact symbols — which is why agents still default to grep. A graph-native server stores deterministic, verifiable edges (calls, implements, references), so it answers "who calls this?" and "what breaks if I change this?" precisely, not probabilistically. Gortex is hybrid — BM25 + semantic + graph-centrality reranking via RRF — so it's graph + vector, not graph vs vector.

## Bottom line

MCP is the socket, not the brain — so the question that matters is what's plugged into it. Score the field against a plain rubric and the picture clears up: claude-context is strong vector search but search-only and needs a key plus a vector DB; Serena is elegant LSP navigation and editing with no key but no semantic search; codanna is a promising lightweight local graph; Context7 solves a different, complementary problem entirely. The gap none of them fills alone is the whole rubric in one install — hybrid graph + vector retrieval, precise callers and usages, sub-millisecond impact, multi-repo contracts, and live freshness, using all three MCP primitives. That's what a graph-native server is for, with the honest caveats that it's a binary-plus-daemon and a couple of refactors are Go-only today. If your agent acts on what it finds, the combination is the trade worth making.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
