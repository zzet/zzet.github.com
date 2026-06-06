---
title: "Your codebase is too large for the context window"
description: "Codebase too large for context window? You can't paste a monorepo, and bigger windows rot. The fix is the right minimal working set, selected on demand."
date: 2026-05-30
lastmod: 2026-06-06
draft: false
slug: "codebase-too-large-for-context-window"
keywords: ["codebase too large for context window", "give Claude whole repo context", "minimal working set context for coding agent", "fit codebase in LLM context window", "context rot in AI coding agents", "large codebase llm context"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You try to give your coding agent the whole repo, and it bounces. The repo is megabytes of source; the window is a few hundred thousand tokens. So you reach for the 1M-token model, the agent crawls and costs more, and the answers somehow get *worse* on the long prompts. That is the wall this post is about: your **codebase is too large for the context window**, it always will be, and the cure is not a bigger window. It is the right minimal working set, selected on demand.

The instinct — "give Claude whole repo context" — is the wrong frame. The repo is orders of magnitude bigger than any window, and the techniques that look like they fit it (dump, embed, summarize) each trade away something you need.

> **Short answer:** A real codebase does not fit, and never will. Anthropic's 1M-token window holds roughly 75,000 lines of code; production monorepos are routinely 10–100× that (the Linux kernel alone is 70,000+ files). Bigger windows also degrade: research on context rot and "lost in the middle" shows accuracy drops as input grows, even when the window isn't full. The durable fix is to stop trying to fit the repo and instead select the smallest high-signal working set on demand — the target symbols plus their real dependencies — under a token budget.

---

## Why your codebase will never fit the context window

Start with the arithmetic, because it ends the debate. Anthropic's own [1M-token announcement](https://claude.com/blog/1m-context) says a million tokens can process "entire codebases with over 75,000 lines of code in a single request." That is the largest mainstream window today, and 75k LOC is a medium service, not a system. The Linux kernel is 70,000+ files. A typical company monorepo is hundreds of thousands to millions of lines. The repo is not 1.2× too big for the window — it is 10–100× too big. No near-term window closes that gap.

So the first correction: **fitting the whole repo is not the goal.** It is not achievable, and chasing it wastes effort that should go into selection.

The second correction is subtler, and it kills the "just use a bigger window" reflex. Even when the repo *does* fit, more context hurts past a point.

**Context rot.** Chroma's [context-rot research](https://www.trychroma.com/research/context-rot) tested 18 models — including Claude Opus 4, Sonnet 4 and 3.5, the GPT-4.1 family, Gemini 2.5, and Qwen3 — and found models do not use long context uniformly. Performance grows "increasingly unreliable as the input length grows," even on simple tasks, and irrelevant "distractor" text hurts more the longer the input gets. The window not being full does not save you; the *amount* of input is itself a quality variable.

**Lost in the middle.** The [Liu et al. paper (TACL 2024)](https://arxiv.org/abs/2307.03172) found LLM accuracy is highest when the relevant fact sits at the beginning or end of the context and degrades significantly when it lands in the middle — even for models explicitly built for long context. Dump a repo and the function the model needs is, by definition, somewhere in the middle.

**The cost shape (stated precisely).** It is tempting to say "long context is expensive," but that is now partly stale: Anthropic removed the long-context surcharge and made the 1M window generally available at standard per-token rates ([reported around March 2026](https://thenewstack.io/claude-million-token-pricing/)) — a 900k-token request bills at the same per-token rate as a 9k one. So the durable cost argument is not a premium band. It is **linear per-token cost across many turns**: a fat prompt is re-billed on every subsequent message of the session, so a dumped repo you carry through 40 turns is paid for 40 times.

Put together: the repo can't fit, and even where it can, a bigger window raises the ceiling without stopping quality from rotting as you fill it. This is the same wall described from the cost side in [your AI agent wastes most of its tokens finding code](/gortex/ai-agent-token-waste-code-retrieval/).

### Adopt the frame the model vendor already uses

You do not have to take this on faith from a tool vendor. Anthropic's [effective context engineering guide](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) defines the goal as finding "the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome," and says context "must be treated as a finite resource with diminishing marginal returns." It explicitly advocates a *just-in-time* approach: load data at runtime via lightweight identifiers (file paths, queries, links) rather than pre-loading everything. The model vendor's own engineering team argues for curation over dumping. That is the thesis of this post.

---

## The honest menu: how to fit a codebase in an LLM context window

If you can't paste the repo, you select from it. There are four real families of approaches, each with genuine strengths and a real limit. A reader who picks any of them should leave smarter.

| Approach | What it does | Strength | Limit |
|---|---|---|---|
| **Whole-repo packing** | Concatenate the repo (or a compressed digest) into one file | Zero infra, one-shot, full text | Static snapshot; re-bills every turn; still capped by the window |
| **Vector RAG** | Embed AST chunks, retrieve by similarity on demand | Strong NL recall at scale; incremental | Approximates relationships; can't follow code edges; chunk-size trade-off |
| **Repo map (ranked)** | Send a token-budgeted map of key symbols + signatures | Whole-repo structure cheaply; real graph ranking | A static digest, not the actual transitive dependencies of *this* task |
| **Long-context model** | Put a lot in-context with a 1M window | Real, large, simple when you genuinely want a lot in-context | 75k-LOC ceiling; context rot; linear per-turn cost |

A few of these deserve names, because they are the strongest examples in their category.

**Whole-repo packing — Repomix.** [Repomix](https://github.com/yamadashy/repomix) packs an entire repository into a single AI-friendly file with per-file and total token counts, plus a `--compress` mode that uses Tree-sitter to extract key code elements and cut the token count. It is the cleanest way to hand a chat window a small repo. The structural limit is in the name: a *pack* is a static snapshot that re-bills on every turn and cannot follow `handler → service → repo` on demand. The deeper trade-off is in [beyond Repomix and Gitingest — query a graph, don't concatenate](/gortex/repomix-gitingest-alternative-graph/).

**Vector RAG — claude-context.** [claude-context](https://github.com/zilliztech/claude-context) (MIT, ~11.7k stars as of mid-2026) is an MCP server whose pitch is "make the entire codebase the context for any coding agent." It indexes the repo into Milvus/Zilliz and runs hybrid BM25 + dense-vector search on demand instead of stuffing directories into the prompt. Real strengths: AST-aware (tree-sitter) chunking, Merkle-tree incremental re-indexing, and multiple embedding providers (OpenAI key by default; VoyageAI, Gemini, and local Ollama supported). Its on-demand premise is correct. The limit is *what* it retrieves — covered below. The commercial version of this approach is [Cursor's server-side index](https://cursor.com/blog/secure-codebase-indexing): a Merkle tree of file hashes, AST chunks, embeddings in Turbopuffer, re-synced roughly every few minutes (per Cursor's published description as of 2026).

**Repo map — Aider.** [Aider's repository map](https://aider.chat/docs/repomap.html) is the closest mainstream precedent to a graph approach, and deserves direct credit. It sends a concise, token-budgeted map of the whole git repo — key symbols and signatures — ranked by a PageRank-style algorithm over a file-dependency graph, so the model sees structure without every file. It defaults to about 1k tokens via `--map-tokens` and expands when no files are in the chat. It is mature (roughly 45k stars and Apache-2.0 as of 2026) and uses a *real graph ranking*, not similarity. Its limit is that the map is a static signature digest of the whole repo — it shows you the important symbols, but it does not assemble the *actual transitive dependencies of the task in front of you* under a budget.

**Long-context model — Claude 1M.** The honest position is that this helps and is not the enemy. When you genuinely want a lot in-context — a self-contained service, a focused investigation — a 1M window is real and useful. Concede it plainly. Then remember the ceiling (75k LOC vs a real monorepo), the rot (Chroma, Liu et al.), and the linear per-turn cost. Long context and a curated working set are complementary, not rivals: the best move is to feed a *curated* set into a capable model, long-context or not.

### Why code specifically breaks naive RAG

There is one failure that matters more than the others for code, and it is why the vector path needs care. **Code is a dependency graph, not prose.** Semantic similarity finds chunks that *read like* your query; it cannot guarantee it retrieves the definition, the callers, or the interface a function actually depends on. As [MindStudio puts it](https://www.mindstudio.ai/blog/is-rag-dead-what-ai-agents-use-instead), a chunk calling `processPayment()` will not necessarily retrieve the chunk *defining* it — and chunking forces an unsolvable trade-off: too small and you drop context, too large and you add noise.

The consequence is worse than an incomplete answer. A missed definition makes the agent **confidently write broken code** — it invents a signature for the function it never saw. That is a correctness failure, not just a recall failure, and it is the same root cause behind [hallucinated code](/gortex/stop-ai-agent-hallucinating-code/). The function on line 200 that needs the interface on line 10 of another file is precisely what similarity retrieval cannot reliably stitch together.

---

## The minimal working set: select from the dependency graph

The reframe is now concrete. The cure for context rot is not a bigger window — it is a *smaller, correct, graph-selected* window. A **minimal working set** is the smallest collection of code the model needs to do the current task correctly: the target symbols plus their *real* dependencies, assembled on demand under a token budget, instead of pasting the repo. You build it by selecting from the codebase's dependency graph — pick the task-relevant seeds, walk their transitive call/import/implements closure, then compress the lower-priority bodies to signatures. This is the automated form of [context engineering for coding agents](/gortex/context-engineering-for-coding-agents/).

[Gortex](https://github.com/zzet/gortex) is the deterministic backend for that selection. It indexes a repo into an in-memory [code knowledge graph for LLMs](/gortex/code-knowledge-graph-for-llms/) and serves it over MCP, CLI, and a web UI — Apache 2.0, 257 languages, 100+ MCP tools. The graph holds real `calls`, `implements`, and `references` edges, so the working set is built from actual dependencies, not approximate neighbors. The relevant tools:

- **`smart_context`** returns a task-aware minimal working set in **one call that replaces 5–10 exploration reads**, with a `blast_radius` block and a file-clustered set. The seed count and token budget scale to the size of the graph, so the same call behaves on a 5k-node repo and a 1.6M-node one.
- **`context_closure`** walks the *transitive* call / implements / reference closure under a single token budget. This is the part RAG can't do: the function on line 200 *and* the interface on line 10 it depends on, pulled together because the edge between them is real.
- **`compress_bodies`** elides function bodies to signatures plus doc-comments — landing at **~30–40% of original tokens** across 14 languages — so you spend full fidelity on the symbols that matter and skeletons on the rest. Fidelity is graded, not all-or-nothing.
- **Delta packing** (`delta_from` / `if_none_match` ETag dedup) sends only what changed between turns, instead of re-paying for the whole working set on every message — directly attacking the linear per-turn cost.

The before/after is measurable. On Gortex's own benchmark (`ripgrep + full read → gortex`):

| Query | ripgrep + full read | gortex | recall@2k (rg → gortex) |
|---|---|---|---|
| `AddObservation` | 31,530 tokens | **972** | 0.00 → **1.00** |
| `IsSymbolQuery` | 23,027 tokens | **577** | 0.00 → **1.00** |
| `FileCoherenceSignal` | 14,268 tokens | **151** | — |
| `alphaFuse` | 14,574 tokens | **534** | — |

The headline is **median recall@2k = 1.00 vs 0.00 for ripgrep**, at **3–50× fewer tokens per response** (the 50× is the per-response peak on identifier lookups, not an average). The recall figure is the one that matters here: inside a 2k-token budget, ripgrep returned the *wrong* thing on these queries; the graph returned the right thing in a fraction of the tokens. That is exactly the budget regime a minimal working set lives in.

For scale context: Gortex indexes [torvalds/linux](https://github.com/torvalds/linux) (70,333 files, 1.69M nodes, 6.24M edges) in about 3 minutes and holds the graph in ~5 GB of RAM. The repo that *can't* fit a context window fits a graph you query on demand.

### Graph and vector, not graph vs vector

One correction worth making, so this doesn't read as anti-RAG. Gortex retrieval is **hybrid**: BM25/FTS5 lexical search, default-on baked GloVe-50d embeddings (zero deps, no API key), Reciprocal Rank Fusion to blend them per query, and graph-centrality reranking on top. It keeps the semantic recall vector tools are good at, adds deterministic graph edges that embeddings only approximate, and needs no external vector DB. The default GloVe embeddings are lightweight; for stronger semantic recall you opt into MiniLM, Ollama, or OpenAI. The deeper split is in [graph RAG vs vector RAG for code](/gortex/graph-rag-vs-vector-rag-for-code/).

### Where this doesn't help (the honest concession)

For a small repo, none of this is worth it — a 1M window or even a Repomix pack will swallow a 75k-LOC service whole, and standing up a daemon is overkill. The graph approach pays off as repo size and session length grow, because both compound the cost of dumping. Gortex is also a binary plus a daemon (heavier to bootstrap than a one-shot CLI dump, no in-browser story), its default GloVe embeddings trade recall for zero setup, and `move_symbol` / `inline_symbol` are Go-only for now. And it does not replace a capable model — it feeds one. The honest end state: a curated, graph-selected working set fed into a capable (even long-context) model beats both a dumped repo and a window-less guess.

---

## FAQ

**Can I fit my entire codebase in an LLM context window?**
Almost never for a real project. The largest mainstream window today is about 1 million tokens, which Anthropic says holds roughly 75,000 lines of code in a single request. Production monorepos are routinely 10–100× larger (the Linux kernel alone is 70,000+ files), so the repo simply does not fit. The practical answer is not to fit the whole repo, but to select the right minimal working set on demand.

**Does a bigger context window fix the problem?**
It helps for some tasks but does not solve it. Research on context rot (Chroma tested 18 models) and the "Lost in the Middle" paper both show LLM accuracy degrades as input grows longer, and information in the middle of a long prompt is recalled worst. Anthropic's own guidance treats context as a finite resource with diminishing returns and recommends the smallest set of high-signal tokens, not the largest. A bigger window raises the ceiling but does not stop quality from rotting as you fill it.

**What is context rot in AI coding agents?**
Context rot is the measurable drop in an LLM's output quality as the amount of input grows, even when the window is not full. Chroma's 2025 study found models do not use context uniformly: performance becomes increasingly unreliable as input length increases, and irrelevant "distractor" text hurts more at longer lengths. For coding agents this means a repo dumped into the prompt can actively degrade answers, not just cost more.

**Why does vector search (RAG) miss things in code?**
Code is a dependency graph, not prose. Semantic similarity finds chunks that read like your query, but it can't guarantee it retrieves the definition, callers, or interface a function actually depends on — a function on line 200 may need an interface defined elsewhere that no embedding links to it. Chunking forces a lose-lose trade-off (too small drops context, too large adds noise), and a missed definition makes the agent confidently write broken code. Graph-aware retrieval follows real call/implements/reference edges instead of approximating them.

**What is a minimal working set for a coding agent, and how do I build one?**
A minimal working set is the smallest collection of code the model needs to do the current task correctly — the target symbols plus their real dependencies — assembled on demand under a token budget instead of pasting the repo. You build it by selecting from the dependency graph: pick the task-relevant seeds, walk their transitive call/import/implements closure, then compress lower-priority bodies to signatures. Gortex does this directly: `smart_context` returns a task-aware working set, `context_closure` walks the transitive closure under one token budget, and `compress_bodies` plus delta packing keep token spend low and send only what changed between turns.

---

## Bottom line

The repo is too big for the window, and it always will be — 1M tokens is ~75k LOC against monorepos 10–100× that. A bigger window does not rescue you: context rot and lost-in-the-middle degrade answers as input grows, independent of price, and the durable cost is linear per-token billing across every turn you carry a fat prompt. The menu is real — pack with Repomix, retrieve with vectors, map with Aider, or reach for a long-context model — and each has its place. But the answer is not more context, it is the *right* minimal working set: the target symbols plus their real dependencies, selected from the dependency graph on demand, under a token budget. Curate, then feed a capable model. For a small repo, skip all of it; for anything that grows, the working set beats the dump.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
