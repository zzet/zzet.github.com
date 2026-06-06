---
title: "Context engineering for coding agents, automated"
description: "Context engineering for coding agents, explained: the working-set and just-in-time-retrieval framing, and how a code graph automates the retrieval slice."
date: 2026-05-19
lastmod: 2026-06-06
draft: false
slug: "context-engineering-for-coding-agents"
keywords: ["context engineering for coding agents", "smart context one call working set", "AI agent re-explores codebase every session", "AI coding agent forgets project", "automated context engineering", "context engineering vs prompt engineering"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

Your coding agent opens a fresh session and re-discovers your codebase from scratch. It greps for the handler, reads three files to find the service, traces a call by hand, and forgets all of it the moment the window clears. Next session, same archaeology. You end up hand-feeding it files — pasting paths, telling it where the constraint lives — because you are the only one who remembers. That manual feeding is **context engineering for coding agents**, done by hand every turn. This post is about the part of it you can automate.

The term is the rising successor to "prompt engineering," and most of the canonical writing on it is theory — working set, attention budget, just-in-time retrieval, persistent memory. The theory is correct. What is missing is a deterministic way to *compute* the working set instead of curating it by hand.

> **Short answer:** Context engineering is deciding what goes into the model's context window at each step — Anthropic calls it "curating and maintaining the optimal set of tokens during LLM inference" and "the natural progression of prompt engineering." For a coding agent that means choosing the files, symbols, call chains, and project memories the model sees, aiming for "the smallest possible set of high-signal tokens." The retrieval slice of that work — assembling the right working set and surfacing prior knowledge — is the part a code graph can automate. The rest (tool design, instructions, examples) still needs human curation.

---

## What is context engineering, and how is it different from prompt engineering?

Prompt engineering is about *how you ask*: wording, instructions, few-shot examples. Context engineering is about *what the model knows, sees, and remembers as it acts* — retrieved code, tool outputs, prior steps, the project's rules. Anthropic's [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (Sept 29, 2025) frames it as "the natural progression of prompt engineering": it does not replace prompts, it absorbs them into a larger system that also manages memory and retrieval.

The reason the discipline exists is physical. Anthropic describes a finite "attention budget" depleted by every token, and **context rot**: "as the number of tokens in the context window increases, the model's ability to accurately recall information decreases." More context is not more knowledge past a point — it is more noise. The goal is "finding the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome."

For coding specifically, Birgitta Böckeler's [Context Engineering for Coding Agents](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html) (ThoughtWorks, on martinfowler.com, Feb 5, 2026 — by Böckeler, not Martin Fowler) defines it as "curating what the model sees so that you get a better result," organized into three categories: **reusable prompts** (instructions, rules — your `CLAUDE.md` / `AGENTS.md`), **context interfaces** (tools, MCP servers, skills), and **workspace files** (file reading and searching, "the most basic and powerful context interfaces"). That third category is where the daily pain lives, and it is the slice this post is about.

---

## The mental model: window as RAM, repo as disk

Use the operating-system analogy. The **context window is RAM**: fast, finite, wiped between sessions. Your **repository is disk**: durable, large, not directly addressable. Context engineering is the memory manager deciding what to page in, when, and what to evict. Four consequences fall out, mapping exactly to the canonical solution shape:

| Problem | OS analogy | The technique |
|---|---|---|
| The whole repo can't fit | Disk is bigger than RAM | **Working set** — load only the symbols this task touches |
| You don't know up front what's needed | Demand paging | **Just-in-time retrieval** — fetch on demand via lightweight identifiers |
| RAM is wiped between runs | Persist to disk | **Persistent memory** — store learnings outside the window |
| RAM fills mid-task | Swap / GC | **Compaction** — summarize and reclaim space without losing the thread |

Anthropic's guide names the same four: just-in-time retrieval ("maintain lightweight identifiers and dynamically load data into context at runtime," which "mirrors human cognition"), compaction, and structured note-taking. The repo will never fit the window, so the work is selection, not packing — the wall covered from the sizing angle in [your codebase will never fit the context window](/gortex/codebase-too-large-for-context-window/).

---

## Why hand-assembling context doesn't scale

When you paste paths and tell the agent which file holds the constraint, you are doing graph traversal by hand. You are the index — you walk `handler → service → repo`, you remember where the rate limiter lives, you recall the migration that changed the schema. The agent can't, so you do.

[LogRocket's write-up on context engineering for IDEs](https://blog.logrocket.com/context-engineering-for-ides-agents-md-agent-skills/) documents the cost: agents re-discover and re-explain the codebase each session, burning tokens, slowing execution, and causing mistakes when a constraint is buried. That splits into two failure modes — the agent **re-explores the codebase every session** (re-deriving architecture it already learned, because RAM was wiped) and **forgets the project between sessions** (the "validate at the boundary, never in the handler" rule you explained yesterday is gone today). Both are mechanical retrieval-and-memory problems that should be automated. (The deeper cost breakdown is in [your AI agent wastes most of its tokens finding code](/gortex/ai-agent-token-waste-code-retrieval/); the scaling-wall version is [why coding agents fail past 400k lines](/gortex/why-ai-coding-agents-fail-large-codebases/).)

---

## The honest menu: how teams automate context selection today

The real options, each with a strength and a limit:

| Approach | What it automates | Strength | Limit |
|---|---|---|---|
| **Manual curation** | Nothing — you do it | Maximum control; zero infra | You are the index; doesn't survive sessions |
| **Vector index** (Cursor, claude-context) | Retrieval by similarity | Strong NL recall; incremental; privacy-conscious server-side option | Approximates relationships; can't follow code edges deterministically |
| **Read-on-demand memory files** (Serena) | Persistent memory | Free, LSP-accurate, no embeddings needed | Memories are read when asked, not surfaced automatically |
| **Agent-mode file tools** (Continue) | Retrieval via search tools | Open-source, integrated, no separate index to stale | Still per-file search; no graph edges or auto-memory |
| **Code graph** (Gortex) | Working set + JIT + auto-memory | Deterministic edges; auto-surfaced memory | Retrieval slice only; broader CE is still on you |

A few deserve names. **[Serena](https://github.com/oraios/serena)** (oraios) is the closest comparison on persistent memory: an LSP-backed semantic MCP coding toolkit whose onboarding pass writes Markdown memories to `.serena/memories/` that the agent reads on demand. It is genuinely strong — as of 2026, MIT-licensed and free, ~25k GitHub stars, v1.5.3 (May 2026), no embeddings or API key needed (verify current figures on its repo). The limit is that memories are *read when the agent thinks to ask*; an open feature request ([issue #994](https://github.com/oraios/serena/issues/994)) asks for semantic memory search precisely because flat files don't scale to "surface the relevant memory automatically."

**[Cursor's codebase index](https://cursor.com/blog/secure-codebase-indexing)** is the privacy-conscious vector approach done well — Merkle hashes, syntactic chunks, embeddings in Turbopuffer, no raw code retained — but its limit is what embeddings express: chunk similarity approximates relationships rather than following the actual `calls`/`implements`/`references` edges. **[Continue.dev](https://docs.continue.dev/reference/deprecated-codebase)** changed its mind: as of 2026, its `@codebase` provider is deprecated "in favor of a more integrated approach to codebase awareness," replaced by Agent-mode file search plus MCP servers — the industry moving from a static index toward on-demand retrieval. (For completeness, as of 2026 [Sourcegraph Cody](https://sourcegraph.com/blog/changes-to-cody-free-pro-and-enterprise-starter-plans) discontinued its Free tier in July 2025 and is positioned as enterprise-only.)

The pattern: everyone agrees on the theory. The gap is computing the working set deterministically and surfacing memory automatically.

---

## Automated context engineering with a code graph

A code graph indexes your repository into nodes (symbols, files) and typed edges (`calls`, `implements`, `references`). Once those edges exist, "the smallest high-signal set for this task" becomes a graph query, not a judgment call. That is [Gortex](https://github.com/zzet/gortex): an in-memory code graph over an MCP server, indexing **257 languages** through **100+ MCP tools**, Apache-2.0 licensed. It automates the retrieval slice — here is the mapping onto the canonical solution shape.

**Working set — `smart_context` (one call).** Instead of 5–10 manual exploration reads, one call returns a task-aware minimal working set: the target symbols, their real callers and callees, file-clustered, with graded fidelity and an upfront token estimate. This is Anthropic's "smallest possible set of high-signal tokens," *computed from edges* rather than assembled by hand.

```jsonc
// one call replaces 5-10 manual exploration reads
smart_context({ "task": "add rate limiting to the login handler" })
// -> target + callers + callees, file-clustered, with a token estimate
```

**Just-in-time retrieval — `prefetch_context`.** The graph predicts the symbols the next step is likely to need and stages them — "load on demand via lightweight identifiers," automated from the call structure rather than guessed.

**Persistent memory — `store_memory` / `surface_memories`.** The answer to "my agent forgets the project." Cross-session memories are stored with anchor symbols, and `surface_memories` brings them back **automatically when those anchor symbols enter the working set** — not read-on-demand. Where Serena's `.serena/memories` wait for the agent to open them, a graph-anchored memory appears the moment you touch the code it's about: the "validate at the boundary" rule resurfaces the next time the handler enters context, without anyone asking.

**Compaction survival — session notes and a PreCompact hook.** Session notes (`save_note` / `distill_session`) persist outside the window, and a PreCompact hook injects an orientation snapshot before the window is compacted — so the agent re-orients from a distilled summary instead of restarting blind.

The before/after, concretely:

| Step | Hand-curated context | Graph-automated |
|---|---|---|
| Find the working set | grep + read 5–10 files; you pick | `smart_context` — one call, edges resolved |
| Recall a project rule | you remember and paste it | `surface_memories` — auto on anchor symbol |
| Survive a window compaction | re-explain from scratch | PreCompact orientation snapshot |
| Token cost of retrieval | whole files, re-read each turn | symbol-level, **3–50× fewer per response** |

Stated honestly: **3–50× fewer tokens per response** versus naive file reads, with the ~50× peak on identifier lookups — not a flat 50× — and `get_symbol_source` alone returns ~80% fewer tokens than reading a whole file. What a graph buys architecturally is in [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/).

### Why deterministic edges beat approximate similarity

A graph produces a *correct* working set, not a plausible one, because its edges are verified, not inferred. A vector index returns chunks that look similar; it cannot guarantee it found every caller, because "caller" is structural and embeddings only approximate it. To be fair, Gortex is hybrid, not graph-versus-vectors: it fuses lexical (BM25) and semantic search with graph-centrality reranking, defaulting to a lightweight baked GloVe model (zero setup, no API key) with stronger MiniLM / OpenAI embeddings opt-in. Semantic recall on fuzzy concept queries is modest; the graph's strength is exact, structural retrieval — the deeper comparison is [graph RAG vs vector RAG for code](/gortex/graph-rag-vs-vector-rag-for-code/).

---

## The honest concession: retrieval is one slice

Context engineering is broader than retrieval, and a code graph does not cover all of it. Mapped onto Böckeler's three categories, a graph automates the **workspace files** slice — assembling the working set and surfacing memory. It does not write your **reusable prompts** (instructions, rules, examples), and it is only one of your **context interfaces**. Tool design, the wording of your `CLAUDE.md`, the sub-agent architecture — all of that is still human work, and the larger part of the discipline. What a graph removes is the mechanical part — the by-hand traversal, the re-exploration every session, the forgotten project rules — lookups, not judgment calls. Other limits worth naming: `move_symbol` / `inline_symbol` refactors are Go-only for now, and benchmarks are single-machine figures that vary 2–5× by hardware (`gortex bench` reproduces them).

---

## FAQ

### What is context engineering for coding agents?

Deciding what goes into the model's context window at each step. Anthropic defines it as "curating and maintaining the optimal set of tokens during LLM inference" — for a coding agent, choosing which files, symbols, call chains, and project memories the model sees so it gets the smallest high-signal working set instead of the whole repo.

### Context engineering vs prompt engineering — what's the difference?

Prompt engineering is about *how you ask* (wording, instructions, examples). Context engineering is about *what the model knows, sees, and remembers as it acts* — memory, tool outputs, retrieved code, prior steps. Anthropic calls it "the natural progression of prompt engineering": it absorbs prompts into a larger system rather than replacing them.

### Why does my AI coding agent re-explore (or forget) the codebase every session?

The context window is cleared between sessions, like RAM. Without persistent memory or automated retrieval, the agent re-discovers your architecture every time — costing tokens, wasting turns, and missing buried constraints. The fixes are persistent project memory plus just-in-time retrieval, so the right code loads automatically instead of being re-derived.

### How do you automate context engineering for a coding agent?

Automate the retrieval slice with a code graph. A single call returns a task-aware working set — Gortex's `smart_context` replaces 5–10 manual exploration reads — `prefetch_context` predicts the next symbols, and cross-session memories surface automatically when their anchor symbols enter the set. Retrieval and memory are what a graph automates; tool design and instructions still need human curation.

### Is context engineering just retrieval (RAG)?

No. It also covers tool design, instructions and rules, examples, compaction, and sub-agent architecture — Böckeler's three categories. Retrieval (assembling the right working set and persistent memory) is the slice a code graph automates deterministically; the rest is harness and instruction design.

---

## Bottom line

Context engineering is the right frame: the window is finite RAM, the repo is disk, and the job is selecting the smallest high-signal working set, paging it in on demand, and remembering across sessions. What a code graph adds is automation of the retrieval slice: `smart_context` computes the working set from real edges instead of you traversing it, `surface_memories` brings back project knowledge the moment its anchor symbols enter context, and a PreCompact snapshot keeps the agent oriented through compaction. It is not all of context engineering — tool design and instructions are still your job — but it removes the mechanical part you re-pay by hand every session.

One install configures every detected agent — Claude Code, Cursor, Cline, Copilot, Continue, and more — Apache-2.0, no API key to start: [github.com/zzet/gortex](https://github.com/zzet/gortex).
