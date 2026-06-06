---
title: "Token Efficient Code Retrieval for AI Agents: Stop Wasting Tokens Finding Code"
description: "Token efficient code retrieval for AI agents: why Claude Code is so expensive, the honest fix menu, and how a code graph cuts agent token waste 3–50×."
date: 2026-05-20
lastmod: 2026-06-06
draft: false
slug: "ai-agent-token-waste-code-retrieval"
keywords: ["token efficient code retrieval for AI agents", "reduce Claude Code token usage", "AI agent wastes tokens finding code", "read one function instead of whole file", "why is Claude Code so expensive", "cut agent token waste", "ai coding agent cost"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You ask your agent a one-line question — "what does `AddObservation` do, and who calls it?" — and watch it grep, open a 500-line file to read one 20-line function, open four more candidate files to be sure, and then keep all of them in context for the rest of the session. The answer needed maybe 800 tokens. The agent spent twelve thousand. That gap is the subject of this post: **token efficient code retrieval for AI agents**, and why most of an agent's bill is spent finding code rather than reasoning about it.

This is the mechanism behind "why is Claude Code so expensive." The model is not the waste. The exploration is.

> **Short answer:** AI agents burn tokens because they read whole files to find one function, re-read those files on every subsequent turn, and have no working-set memory. The fix is symbol-level retrieval: fetch one function instead of a 500-line file. Anthropic's own cost docs say "a single go-to-definition call replaces what might otherwise be a grep followed by reading multiple candidate files." A code graph applies that to every read — typically ~80% fewer tokens per symbol, and one `smart_context` call in place of 5–10 exploration reads.

---

## Why your AI agent wastes tokens finding code

Developers describe the pain the same way everywhere: the agent reads ~25 files to answer a question about 3 functions; a 12,000-token question whose real answer needed ~800. (Those are developer observations, not lab measurements — treat them as anecdote.) The shape is right, and four mechanisms produce it.

**File-granular reads.** The smallest unit your agent can read is a file. To use one function it pays for the whole file around it — imports, the other twelve functions, the comments. If the function it wants is 4% of the file, 96% of that read is noise.

**Repeated exploration.** Agentic search means grep, read candidates, grep again. Each step is a fresh round of file reads, and a vague prompt makes the agent scan broadly — Anthropic notes ["token costs scale with context size."](https://code.claude.com/docs/en/costs)

**No working-set memory.** Across turns, and especially across sessions, the agent forgets what it learned about the repo and re-discovers it from scratch. No persistent symbol map means every session re-pays the exploration tax.

**Per-turn re-billing.** This is the quiet killer. Every file the agent read stays in the context window and is **billed again on each subsequent turn.** Anthropic puts it plainly: ["stale context wastes tokens on every subsequent message."](https://code.claude.com/docs/en/costs) Read five 500-line files early in a session and you carry ~2,500 lines of mostly-irrelevant code through every later message.

The bill is real. Anthropic reports average Claude Code spend around **$13 per developer per active day** and **$150–250 per developer per month**, and notes that agent teams in plan mode can use roughly **7× more tokens** because each teammate gets its own context window. Multiply file-granular waste across a team and it stops being a rounding error.

This is the same wall you hit from the other direction when [your codebase is too large for the context window](/gortex/codebase-too-large-for-context-window/): you can't fit the repo, and you can't afford to keep re-reading slices of it either.

---

## The honest menu for token efficient code retrieval for AI agents

There is no single switch. Here's the menu, cheapest-first, with what each one actually buys you.

| Approach | What it does | Strength | Cost / limit |
|---|---|---|---|
| **Context hygiene** | `/clear` between tasks, specific prompts, plan mode, lean `CLAUDE.md` | Free, immediate; kills the worst stale-context re-billing | Manual discipline; doesn't fix the per-read waste |
| **Output compaction / hooks** | Filter verbose tool output with hooks; isolate noisy work in subagents | Keeps junk out of the main context window | Setup effort; doesn't reduce what's read to begin with |
| **Repo / symbol maps** | A persistent codemap so the agent stops re-discovering the repo | Cuts repeated exploration; symbol-level reads | Map can go stale; coverage varies by tool/language |
| **RAG / semantic index** | Embed chunks, retrieve by similarity | Strong natural-language recall at scale | Needs an embedding provider + vector DB; approximate, can go stale |
| **Code knowledge graph** | Deterministic call/reference/implements edges + hybrid search | Exact symbol retrieval + structure on demand | A binary/daemon to run; heavier to bootstrap than a one-shot dump |

A few of these deserve names.

**Repomix** ([code compression docs](https://repomix.com/guide/code-compress)) packs your whole repo into one AI-friendly file, with built-in token counting and a `--compress` mode that uses Tree-sitter to keep signatures, interfaces, types, and class structure while stripping bodies. It's MIT-licensed (as of 2026), zero-infra, and ideal for one-shot whole-repo prompts in a chat window. The official docs label `--compress` experimental and publish *no* token-reduction percentage — third-party blogs estimate around 70%, but that figure is unconfirmed. The structural limit: a packed file is a static snapshot that re-bills on every turn and can't follow `handler → util → lib` on demand. The deeper trade-off is covered in [beyond Repomix and Gitingest — query a graph, don't concatenate](/gortex/repomix-gitingest-alternative-graph/).

**tokenmax-mcp** ([GitHub](https://github.com/justinjamesmathew/tokenmax-mcp)) is the closest cousin to a graph approach: a persistent symbol-level codemap so the agent stops re-discovering the repo. Their numbers are instructive — a 508-file TypeScript repo (~380k tokens of source) compresses to a ~28k-token codemap (~13× smaller), and `read_section` returns one symbol (~5–10× smaller per call). It's MIT-licensed (as of 2026). Two caveats they publish themselves: it's **TypeScript/JavaScript only**, and in fully-agentic single-task bug-fix benchmarks it cost *more* tokens on all three scopes — it's tuned for interactive multi-session work, not autonomous one-shot runs.

**claude-context** ([GitHub](https://github.com/zilliztech/claude-context)) brings the vector-RAG approach: hybrid BM25 + dense vectors, AST chunking, Merkle-tree incremental re-index, and a reported **~40% token reduction at equivalent retrieval quality.** It's MIT-licensed (as of 2026) and among the most drop-in of the vector tools. The cost is infrastructure: it needs an embedding provider (OpenAI / VoyageAI / Ollama / Gemini) *and* a vector DB (Milvus or Zilliz Cloud); the fully-local path is Ollama plus self-hosted Milvus. The commercial polish of this approach is [Cursor's server-side index](https://cursor.com/blog/secure-codebase-indexing) (Merkle tree of file hashes, local chunking, embeddings in Turbopuffer).

### The catch: agentic search already beat vector RAG

Before you reach for a vector index, sit with an uncomfortable result. Anthropic built Claude Code *without* one on purpose. Claude Code creator Boris Cherny: ["Early versions of Claude Code used RAG + a local vector db, but we found pretty quickly that agentic search generally works better."](https://vadim.blog/claude-code-no-indexing) A Claude engineer added that agentic search "outperformed [it] by a lot, and this was surprising." An Amazon Science paper ("Keyword Search Is All You Need") found keyword search via agentic tool use reaches over 90% of RAG-level performance with no vector database. Continue.dev has gone the same way: as of 2026, its `@codebase` provider is [deprecated](https://docs.continue.dev/reference/deprecated-codebase) in favor of agent-mode exploration plus MCP.

So the lesson is *not* "bolt on a stale vector index." Agentic search wins. The problem it leaves unsolved is the one this article is about: the *explore loop itself burns tokens* because every read is file-granular and re-billed. The fix is to keep the agentic loop and make each call cheaper.

---

## Read one function instead of the whole file

The single highest-leverage change is to stop reading files and start reading symbols. Anthropic's cost docs endorse exactly this: code-intelligence tools matter because ["a single 'go to definition' call replaces what might otherwise be a grep followed by reading multiple candidate files."](https://code.claude.com/docs/en/costs) (That endorses the *category* of code-intelligence plugins, not any specific product.)

[Gortex](https://github.com/zzet/gortex) is the deterministic, structure-aware backend for that loop. It indexes a repo into an in-memory [code knowledge graph for LLMs](/gortex/code-knowledge-graph-for-llms/) and serves it over MCP, CLI, and a web UI — Apache 2.0, 257 languages, 100+ MCP tools. It doesn't replace agentic search; it makes each step cost less. The tools that attack token waste:

- **`get_symbol_source`** returns one function, not the file around it — **~80% fewer tokens** than the file read, and it reports `tokens_saved` per call so you can see it.
- **`smart_context`** assembles a task-aware minimal working set in **one call that replaces 5–10 exploration reads**, with a `blast_radius` block and file-clustered context.
- **`compress_bodies`** elides function bodies to signatures plus doc-comments, landing at **~30–40% of original tokens**, across 14 languages — the same signatures-over-bodies idea as Repomix `--compress`, but served per-symbol on demand instead of as a static dump.
- **GCX1 wire format** encodes list-shaped responses at a median **−27.4% tokens** vs JSON at identical fidelity (20/20 round-trip integrity).

Here's the before/after on a real query, from Gortex's benchmark (`ripgrep + full read → gortex`):

| Query | ripgrep + full read | gortex | recall@2k (rg → gortex) |
|---|---|---|---|
| `AddObservation` | 31,530 tokens | **972** | 0.00 → **1.00** |
| `IsSymbolQuery` | 23,027 tokens | **577** | 0.00 → **1.00** |
| `FileCoherenceSignal` | 14,268 tokens | **151** | — |
| `alphaFuse` | 14,574 tokens | **534** | — |

The headline is **median recall@2k = 1.00 vs 0.00 for ripgrep**, at **3–50× fewer tokens per response** (the 50× is the per-response peak on identifier lookups, not an average — most queries land lower). The recall number should stop you: ripgrep returned the wrong thing inside a 2k-token budget on these queries; the graph returned the right thing in a fraction of the tokens.

Because it's a graph, not a snapshot, it also answers the case where a bare snippet *isn't* enough. When you debug cross-cutting logic, `smart_context` and `get_callers` pull the symbol *plus* its callers and callees — exact `calls` / `references` / `implements` edges, not similarity guesses — beating both a grep result and a whole-file dump. That structural recall is also why agents stop falling over when [a codebase grows past 400k lines](/gortex/why-ai-coding-agents-fail-large-codebases/): they navigate by edges instead of re-scanning text.

### Graph + vector, not graph vs vector

One correction worth making: this is not a return to "graph instead of embeddings." Gortex retrieval is hybrid — BM25/FTS5 lexical search, **default-on** baked GloVe-50d embeddings (zero deps, no API key), Reciprocal Rank Fusion to blend them per-query, and graph-centrality reranking on top. So it keeps the semantic recall that vector tools are good at, adds deterministic graph edges that embeddings only approximate, and needs no external vector DB. The default GloVe embeddings are lightweight; for stronger semantic recall you can opt into MiniLM, Ollama, or OpenAI. The split is covered in depth in [graph RAG vs vector RAG for code](/gortex/graph-rag-vs-vector-rag-for-code/).

### Does it actually move the bill?

`gortex savings` prices the difference per-call, per-session, and cumulatively across restarts, against the headline model. One real dashboard reports all-time **93.3% tokens saved**, 11.25M tokens, and **$168.69 in cost avoided** (claude-opus-4, 1,878 calls). That's a single operator's machine over a working period, not a universal guarantee — but it's the right number to watch, because it's measured on your own repo and your own usage.

### Where this doesn't help (the honest concession)

For a tiny repo, plain file reads are fine. Reading a 60-line file to use one function costs almost nothing, and standing up a daemon isn't worth it. The savings track two variables that both compound: **repo size** (how much noise surrounds the symbol you want) and **session length** (how many turns re-bill that noise). On a small repo the math is a wash; on a monorepo across a long session it's the difference between $13 and $1 a day. Gortex is also a binary plus a daemon — heavier to bootstrap than a one-shot CLI dump, with no in-browser story — and `move_symbol` / `inline_symbol` are Go-only for now. Measure on your own repo; don't take a benchmark's word for your numbers.

---

## FAQ

**Why is Claude Code so expensive?**
Cost scales with context size, and the biggest hidden driver is exploration: the agent greps, reads several whole files, and every file stays in context and is re-billed on each turn — Anthropic notes "stale context wastes tokens on every subsequent message." Typical spend is ~$13 per developer per active day and $150–250 per month; agent teams can use roughly 7× more. The fixes: keep context small (`/clear`, specific prompts, plan mode) and use code-intelligence so the agent reads one symbol instead of whole files.

**How do I reduce Claude Code token usage?**
In order of effort: (1) hygiene — `/clear` between tasks, specific prompts, plan mode, lean `CLAUDE.md`; (2) filter verbose output with hooks and isolate it in subagents; (3) give the agent a symbol map so it stops re-discovering the repo; (4) add graph/semantic retrieval so it fetches one function instead of a 500-line file. Anthropic recommends code-intelligence tools because "a single go-to-definition call replaces a grep followed by reading multiple candidate files." A code graph like Gortex returns ~80% fewer tokens per symbol read and can replace 5–10 exploration calls with one.

**Should my AI agent read one function instead of the whole file?**
Yes, whenever the tooling allows it. Reading a 500-line file to use one 20-line function pays for ~480 lines of noise that then sit in context and are re-billed every turn. Symbol-level reads (Gortex `get_symbol_source`, tokenmax `read_section`) return just the function — typically ~80% fewer tokens, often 5–10× smaller per call. The exception is when the agent needs surrounding context, where a graph that pulls the symbol plus its callers/callees beats both a bare snippet and a whole-file dump.

**Is a code knowledge graph better than vector/RAG search for AI agents?**
They solve different halves. Anthropic found agentic search beat their old pre-built vector RAG, and Amazon Science showed keyword + agentic tool use reaches over 90% of RAG performance with no vector DB — but pure agentic search burns tokens re-reading files. A code knowledge graph keeps the agentic loop yet answers from a deterministic graph with real call/reference/implements edges, fusing BM25 + embeddings + graph-centrality reranking. That's "graph plus vector," giving exact symbol retrieval (recall 1.00 vs 0.00 for ripgrep) without an external vector DB or API key.

**How much can token-efficient retrieval actually save?**
It depends on repo size and session length — for a tiny repo plain reads are fine, and savings compound as both grow. claude-context reports ~40% token reduction at equal quality; tokenmax compresses a 508-file repo from ~380k tokens to a ~28k codemap. Gortex's benchmark shows extreme cases dropping from 31,530 tokens to 972, a 3–50× per-response range, and a dashboard reporting 93.3% saved and $168.69 avoided across ~1,900 calls. The catch with any tool: measure on your own repo.

---

## Bottom line

Most of your agent's token bill is finding code, not reasoning about it — file-granular reads, repeated exploration, no working-set memory, and per-turn re-billing of everything it ever opened. Context hygiene and output compaction are free; do them first. Agentic search beats stale vector RAG, so don't reach for an index reflexively. The structural fix is to keep the explore loop and make each call cheaper: read one function instead of the whole file, fetch one minimal working set instead of ten reads, backed by a deterministic graph that remembers what the agent already learned. For a tiny repo, skip all of it. For anything that grows, the savings compound — and they're measurable on your own repo.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
