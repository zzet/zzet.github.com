---
title: "Why AI coding agents fail in large codebases"
description: "Why AI coding agents fail in large codebases: context windows can't hold a monorepo, flattening kills the call graph, and bigger windows rot. Here's the fix."
date: 2026-05-21
lastmod: 2026-06-06
draft: false
slug: "why-ai-coding-agents-fail-large-codebases"
keywords: ["why AI coding agents fail in large codebases", "AI coding agent for monorepo", "cross-repository code navigation", "AI agent loses context in large repo", "coding agent large codebase", "ai agent monorepo", "coding agent 400k lines"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

The agent that aced your weekend side project falls apart on the monolith at work. It hallucinates an interface that doesn't exist, edits the wrong file, forgets the architecture you explained yesterday, and burns thousands of tokens re-reading code it already saw. You tighten the prompt. It doesn't help. The honest reason **why AI coding agents fail in large codebases** is not your prompting — it is architectural, and it gets worse exactly as the repo crosses 400k lines, fans out into a monorepo, or spans several repos at once.

This post is the diagnosis first, then the cure. The diagnosis is not controversial — almost every serious explainer agrees on it. The cure is broadly agreed too: a persistent, queryable graph the agent asks instead of re-reading files. Gortex is one implementation; we'll get there honestly, after the problem.

> **Short answer:** AI coding agents fail in large codebases because the code physically cannot fit the context window (a 50-service monorepo runs to tens of millions of tokens; windows are 100k–2M) and never will. Even the code that *does* fit degrades — "lost in the middle" and "context rot" both show models attend worst to the middle of a long prompt and get less reliable as input grows, well before the window is full. Flattening code into a token stream also destroys the call-graph edges, so the agent re-infers structure from text proximity and hallucinates. And agents are stateless by design, so every session re-pays the exploration tax. Bigger windows do not fix any of this; a persistent, queryable code graph does.

---

## Why AI coding agents fail in large codebases: five mechanisms

There is no single cause. There are five, and they compound. Naming them separately matters, because each one rules out a different "fix" you might otherwise waste a week on.

### 1. The codebase does not fit, and never will

Start with arithmetic. Current LLM context windows run roughly 100k to 2M tokens. A typical 50-service monorepo — shared libraries, IaC, generated types, tests — spans tens of millions of tokens of source. Naive context stuffing degrades around 2,500 files and is financially unsustainable at scale, because every token you cram in is re-billed on every turn ([TianPan.co, Coding Agents in the Monorepo](https://tianpan.co/blog/2026-04-17-coding-agents-monorepo-context-window)). The repo is not 1.2× too big — it is 10–100× too big, and no near-term window closes that gap. This is the wall covered in depth in [your codebase is too large for the context window](/gortex/codebase-too-large-for-context-window/).

### 2. Flattening a graph into a token stream destroys the edges

Code is a graph: functions call functions, types implement interfaces, modules import modules. When you dump it into a prompt, you flatten that graph into a linear sequence. As CloudGeometry puts it: ["When you flatten a graph into a sequence, you lose the relationships. The model can still see the nodes, but it has to reconstruct the edges from text proximity."](https://www.cloudgeometry.com/blog/why-your-enterprise-codebase-will-never-fit-in-a-context-window-and-what-to-build-instead) That reconstruction is guesswork. The agent sees a function and a nearby type and *infers* a relationship that may not exist — which is precisely how an agent invents an interface it never actually saw. The fix is not more text; it is giving the agent the edges directly, the theme of [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/).

### 3. Lost in the middle

Even when the relevant code *does* fit, models don't read it evenly. The [Lost in the Middle paper (Liu et al.)](https://arxiv.org/abs/2307.03172) found accuracy is highest when the needed information sits at the very start or end of the context and degrades sharply when it lands in the middle — with reported drops over 30% on multi-document QA — and this holds *even for models explicitly built for long context*. Dump a repo and the function the model needs is, by definition, somewhere in the middle.

### 4. Context rot

This is the newer, sharper finding, and it kills the "just wait for a bigger window" reflex. Chroma's [Context Rot report (July 2025)](https://research.trychroma.com/context-rot) tested 18 frontier models — GPT-4.1, Claude 4, Gemini 2.5, Qwen3 — and found every one degrades as input length grows, even on simple tasks. A 200k-token model can show significant degradation at 50k tokens. Their phrasing is the point: context rot is **not** context-window overflow. Rot happens well before the window is full. So filling a 1M window with a quarter of your monorepo doesn't just cost more — it makes the answers worse.

### 5. No memory, and cross-repo is a resolution problem

The first four mechanisms are about a single session; large repos also break across sessions and across repos.

**Stateless by design.** Copilot, Claude Code, Cursor, and Windsurf all start each session as a blank slate ([Marvin Lijma, Medium](https://medium.com/@marvin-lijma/why-your-ai-coding-agent-keeps-forgetting-everything-and-why-prompt-engineering-wont-fix-it-a76bdc0a724f)). Explain your architecture for 30 minutes today, face the blank slate again tomorrow. One Microsoft engineer [measured wasting 68 minutes a day](https://devblogs.microsoft.com/all-things-azure/i-wasted-68-minutes-a-day-re-explaining-my-code-then-i-built-auto-memory/) re-explaining code before building persistent-memory tooling. That is hours a week, and it is architectural — the agent throws away the context window when the session ends. No prompt fixes a closed window.

**Cross-repo navigation is a resolution problem.** When one change spans repos, the agent has to follow `go-to-definition` from an import in repo A to a definition in repo B. Sourcegraph's engineers framed this precisely: [the same function name exists in many repos, so resolving a definition requires tracking package name + version as part of the symbol's identity; text matching cannot connect an import in repo A to a definition in repo B](https://sourcegraph.com/blog/cross-repository-code-navigation). That is why they built SCIP. A smarter prompt cannot solve it, because it is not a phrasing problem — it is an identity-resolution problem that needs a semantic index.

---

## Pre-empting the obvious objection: "bigger windows will fix it"

They won't, for two independent reasons — worth being explicit, because this is the first thing every team says.

**Scale.** Enterprise monorepos are tens of millions of tokens across dozens of repos. Stuffing a large fraction of that into every request is financially unsustainable, and you'd re-pay it on every turn. A 10M-token window changes the ceiling, not the economics.

**Quality.** Models do not use long context uniformly. Lost-in-the-middle degrades even explicitly long-context models, and context rot appears well before the window fills — significant degradation at 50k tokens on a 200k-token model. A bigger window is a bigger room with the same bad acoustics in the middle.

| Belief | Reality |
|---|---|
| "A 1M / 10M window will hold my repo" | A 50-service monorepo is tens of millions of tokens across many repos; it still doesn't fit |
| "More context = better answers" | Context rot and lost-in-the-middle degrade quality as input grows, before the window fills |
| "It's a prompting problem" | Statelessness and cross-repo resolution are architectural; no prompt closes a window or resolves a symbol |
| "Vector RAG over the whole repo solves it" | Embeddings approximate relationships; they can't guarantee the *definition*, *callers*, or *interface* a function depends on |

The durable fix is the same one the whole field is converging on: give the agent a queryable, persistent, structured model of the code it asks precise questions of, instead of stuffing files into the prompt. That's [context engineering for coding agents](/gortex/context-engineering-for-coding-agents/) — and it's a category, not a product.

---

## The honest menu: what people actually build for large-repo agents

If raw context stuffing fails, you index. There are a few real families, and a reader who picks any of them should leave smarter.

| Approach | What it is | Strength | Limit |
|---|---|---|---|
| **Repo map (ranked)** | Aider's tree-sitter map: PageRank over symbol defs/refs, packed to a token budget | Whole-repo structure cheaply; real graph ranking | Static signature digest, not the task's actual transitive deps; single-repo |
| **Cloud context engine** | Augment: real-time semantic index / knowledge graph across repos | Built for large codebases; "retrieval beats raw model" thesis is correct | Cloud SaaS; vendor metrics; your code leaves the machine |
| **Enterprise code search** | Sourcegraph/Cody: SCIP-based cross-repo navigation, RAG, governance | The incumbent for org-scale cross-repo navigation and admin | Enterprise-only as of 2026; heavy to run |
| **Local graph daemon** | Gortex: in-process knowledge graph per repo, queried over MCP | Local-first, open-source, deterministic edges, multi-repo | Binary + daemon to bootstrap; no in-browser story |

A few deserve names, because they are the most mature options today and they prove the diagnosis is right.

**Aider's repo map** is the closest mainstream precedent and deserves credit. Its [tree-sitter repository map](https://aider.chat/2023/10/22/repomap.html) uses a personalized-PageRank graph over symbol definitions and references to pick the most relevant code within a token budget (default ~1k tokens). That the most-cited open agent already replaced file-stuffing with a *queryable structural graph* is strong evidence the field has converged on the cure. Its limit: the map is a static whole-repo digest, single-repo, not the actual transitive dependencies of the task in front of you.

**Augment Code** markets a [Context Engine](https://www.augmentcode.com/context-engine) — a real-time semantic index / knowledge graph across repos, services, and history, advertising "1M+ files indexed" and a demo on Elasticsearch's 3.6M Java LOC. The thesis ("context is the difference") is correct, and it is genuinely built for large codebases. Frame it accurately: it is a cloud SaaS context engine, the "1M+ files" figure is vendor marketing rather than an independent benchmark, and your source is indexed off-machine.

**Sourcegraph / Cody** is the enterprise incumbent, and the honest move is to concede it. For org-scale code search and cross-repo navigation, with mature SCIP indexing, governance, and admin, it is the default — and a [self-hosted Sourcegraph and Cody alternative for AI agents](/gortex/sourcegraph-cody-alternative-self-hosted/) is a different category, not a drop-in for enterprise governance. One caution, stated with a date: as of 2026 Cody is enterprise-only. Sourcegraph [removed the Cody Free and Pro tiers on 2025-07-23](https://sourcegraph.com/blog/changes-to-cody-free-pro-and-enterprise-starter-plans) and steers individuals to Amp. Third-party price figures (~$59/user/mo, ~$16K platform deals) are unconfirmed; don't treat them as fact. The practical effect: the individual developer and the small team were handed off — exactly the segment a local, free, open-source tool serves.

---

## How a persistent code graph fixes the large-codebase failure

The cure every page above describes — a queryable, persistent, structured graph the agent asks instead of stuffing files — is what Gortex implements as a local-first, Apache-2.0, agent-native daemon. Each failure mechanism maps to a concrete capability.

**The repo doesn't fit → don't load it; query it.** Gortex indexes a repository into an in-memory knowledge graph and exposes it over CLI, an MCP server, and an HTTP API. The agent calls `search_symbols`, `find_usages`, `get_callers`, or `smart_context` and gets back the few symbols it needs — not megabytes of files. On the retrieval benchmark, an identifier query that costs ripgrep + full-read 31,530 tokens (`AddObservation`) returns in **972** tokens with recall@2k of **1.00 vs 0.00** for ripgrep; `IsSymbolQuery` goes 23,027 → **577**. The headline is **3–50× fewer tokens per response** (the 50× is the per-response peak on identifier lookups, not an average) at **median recall@2k = 1.00 vs 0.00** for ripgrep.

**Flattening kills edges → the edges are first-class.** The graph stores deterministic, verifiable edges — `calls`, `implements`, `references` — that embeddings only approximate. `find_usages` returns every reference with no silent false negatives; `get_call_chain` follows `handler → service → repo` as real edges, not text proximity. The agent reads structure instead of guessing it.

**Lost-in-the-middle / context rot → smaller, sharper prompts.** Because retrieval is precise, the working set stays small — no fat middle to get lost in, far less input to rot. `get_symbol_source` returns one function at ~80% fewer tokens than reading the file; `compress_bodies` elides bodies to signatures at ~30–40% of original tokens.

**No memory → a warm daemon plus cross-session memories.** A long-living daemon holds the live graph for every repo and keeps it warm across sessions (warm-restart reconcile is sub-second; incremental re-index ~200ms vs a 3–5s full re-index). On top of that, cross-session development memories (`store_memory` / `query_memories` / `surface_memories`) let the agent record and recall project knowledge between sessions, so it stops re-paying the 68-minute exploration tax.

**Cross-repo is a resolution problem → multi-repo workspace = N repos in one graph.** Register several repos (`track_repository`) and they live in a single graph with cross-repo symbol resolution and call chains. Gortex also auto-detects and matches provider↔consumer API contracts across repos, normalized to canonical IDs like `http::GET::/api/users/{id}` — covering HTTP routes, gRPC, GraphQL, message topics (Kafka/RabbitMQ/NATS), WebSocket, and env vars — and checks them for orphans and mismatches (`contracts action: check`). That is targeted provider/consumer orphan-and-mismatch detection — it catches "repo A calls an endpoint repo B no longer serves" — not full distributed tracing.

**Does it actually scale past 400k lines?** Yes, reproducibly. Gortex indexes the full Linux kernel — `torvalds/linux`, 70,333 files — into **1,690,174 nodes and 6,239,570 edges in about 3 minutes**, in-process, no DB, no network, in ~5 GB of RAM. microsoft/vscode (10,762 files) indexes in ~1 minute. These are single-machine numbers on Apple Silicon; absolute timings vary 2–5× by hardware, reproducible via `gortex bench`.

```bash
# Index N repos into one graph, then query across them
gortex track ./payments-api
gortex track ./billing-service
gortex track ./shared-protos
```

```text
# "Who consumes an endpoint nobody provides anymore?"
# The agent calls the MCP tool: contracts with action: check
contracts(action: "check")        # orphan providers/consumers across repos

# The agent asks the graph instead of re-reading files
# (via MCP: search_symbols, get_call_chain, smart_context, query_memories)
```

### Graph and vector, not graph vs vector

So this doesn't read as anti-RAG: Gortex retrieval is **hybrid**. BM25/FTS5 lexical search, default-on baked GloVe-50d embeddings (3.8MB embedded, zero deps, no API key), Reciprocal Rank Fusion to blend them per query, and graph-centrality reranking on top. It keeps the semantic recall vector tools are good at, adds deterministic graph edges, and needs no external vector DB. The default GloVe embeddings are lightweight — zero-setup, no API key; for stronger semantic recall you opt into MiniLM, Ollama, or OpenAI. Semantic recall on concept/multi-hop tiers is modest (R@5 ~25–30%); exact-symbol queries are where the lexical+graph side dominates (bm25 R@5 = 96.8%).

### Where this doesn't help (the honest concession)

For a small repo, none of this is worth it — a long-context model or a single Repomix-style pack will swallow a 75k-LOC service whole, and standing up a daemon is overkill. The graph approach pays off as repo size, repo count, and session length grow, because all three compound the cost of dumping. Gortex is a binary plus a daemon (heavier to bootstrap than a one-shot CLI dump, with no in-browser story), `move_symbol` and `inline_symbol` are Go-only for now, and it does not replace a capable model — it feeds one. And for enterprise governance, org-wide admin, and audited cross-repo search at scale, Sourcegraph remains the incumbent; Gortex is the local, free, open-source option, not a replacement for that category.

---

## FAQ

**Why do AI coding agents fail in large codebases?**
Because the codebase physically cannot fit the model's context window and never will: a 50-service monorepo can run to tens of millions of tokens while windows are 100k–2M. Even when relevant code fits, two well-documented effects degrade it — "lost in the middle" (models attend best to the start and end of a long context, much worse to the middle) and "context rot" (Chroma showed all 18 tested frontier models get worse as input grows, with a 200k-token model degrading noticeably by 50k). On top of that, flattening code into a token stream destroys the call-graph edges, so the agent re-infers structure from text proximity and often hallucinates an interface it never actually saw.

**Will bigger context windows (1M, 10M tokens) fix AI coding agents on large repos?**
No. First, the scale gap is structural: enterprise monorepos are tens of millions of tokens across dozens of repos, and stuffing a large fraction of that into every request is financially unsustainable. Second, models do not use long contexts uniformly — lost-in-the-middle degrades even explicitly long-context models, and context rot appears well before the window is full (significant degradation at 50k tokens on a 200k-token model). The durable fix is to give the agent a queryable, structured graph it asks precise questions of, instead of stuffing files into the prompt.

**What is the best AI coding agent setup for a monorepo or polyrepo?**
Don't rely on raw context stuffing. Teams getting reliable cross-service agent work use a persistent, structured index/graph the agent navigates precisely. Cross-repo is specifically a resolution problem — the same function name exists in many repos, so go-to-definition needs package + version identity (this is why Sourcegraph built SCIP). A local-first option is Gortex: a long-living daemon holds a live in-memory graph for every repo, a multi-repo workspace puts N repos in one graph with cross-repo symbol resolution, call chains, and provider/consumer API-contract matching, and it indexed the Linux kernel (1.69M nodes) in about 3 minutes.

**Why does my AI coding agent forget my project every session?**
Because agents are stateless by design — Copilot, Claude Code, Cursor, and Windsurf all close the context window when a session ends, so nothing is written down between sessions. You pay a re-exploration tax every time (one Microsoft engineer measured 68 minutes a day re-explaining code). Prompt engineering cannot fix it; you need persistent memory outside the model. Gortex addresses this with a daemon that keeps the graph warm across sessions plus cross-session development memories the agent can store and recall.

**Sourcegraph/Cody vs Augment vs Gortex for large codebases — what's the difference?**
Sourcegraph/Cody is the enterprise incumbent for org-scale code search and cross-repo navigation (SCIP), but as of 2026 it is enterprise-only — Sourcegraph removed the Cody Free and Pro tiers on 2025-07-23 and steers individuals to Amp. Augment Code is a cloud "context engine" that maintains a real-time semantic index across repos (markets 1M+ files indexed). Gortex is the local-first, Apache-2.0, agent-native option: an in-process knowledge graph with deterministic edges, multi-repo workspaces, cross-repo contract checking, and cross-session memory, with no cloud dependency. Pick Sourcegraph for enterprise governance, Augment for a managed cloud engine, Gortex when you want local, free, and open-source.

---

## Bottom line

Coding agents fail past ~400k lines for five compounding reasons, none of which is your prompting: the repo doesn't fit (tens of millions of tokens vs a 100k–2M window), flattening it destroys the call-graph edges, lost-in-the-middle and context rot degrade even the code that fits, the agent forgets everything between stateless sessions, and cross-repo navigation is a symbol-resolution problem a prompt can't solve. Bigger windows raise the ceiling without fixing the acoustics. The field agrees on the cure — a persistent, queryable, structured graph the agent asks instead of re-reading files — and you have real choices: Aider's repo map for a single repo, Augment as a cloud context engine, Sourcegraph for enterprise governance. Gortex is the local-first, open-source instance of that cure: an in-process graph that holds N repos at once with deterministic edges, cross-repo call chains and contract checks, and cross-session memory — and it indexed the Linux kernel's 1.69M nodes in about 3 minutes. For a small repo, skip all of it; for anything that grows, query a graph instead of stuffing the window.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
