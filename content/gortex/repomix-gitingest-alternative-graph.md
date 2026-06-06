---
title: "Repomix Alternative for AI Agents: Query a Graph, Don't Concatenate"
description: "A Repomix alternative for AI agents: why concatenating a repo blows the token budget, when packers still win, and how a code graph fetches only the working set."
date: 2026-06-02
lastmod: 2026-06-06
draft: false
slug: "repomix-gitingest-alternative-graph"
keywords: ["repomix alternative for ai agents", "repomix vs gitingest", "repomix too many tokens large codebase", "turn github repo into LLM prompt", "code2prompt alternative", "query codebase instead of concatenate"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You point Repomix at a small repo, paste the output into a chat window, and it works beautifully — the model now has the whole project and answers well. So you do the same thing on the real codebase, the one with a few hundred thousand lines, and the output is millions of tokens. It won't fit. You add `--compress`, you add include patterns, you split the file, and now you're hand-curating scope on every run. That is the wall this post is about, and the search behind it: a **Repomix alternative for AI agents** that doesn't blow the token budget the moment the repo stops being small.

The honest version of the answer is not "Repomix is bad." It is excellent at what it does. The point is that packing a whole repo into one prompt and *querying* a repo on demand are two different jobs, and they fail in opposite directions as the repo and the session grow.

> **Short answer:** Repomix, Gitingest, and code2prompt all concatenate — they pack the repo (or a filtered slice of it) into one text dump. That's the right tool for a small repo, a one-shot share, or feeding a model a project it has never seen. It breaks down at scale: even with compression, large monorepos overflow the window, and a flat wall of text gets partly ignored ("lost in the middle"). The alternative is to stop packing: index the repo once into a code graph and fetch only the working set per task. Gortex's `smart_context` and `context_closure` do that — 3–50× fewer tokens per response, with deterministic call/reference edges instead of a static dump.

---

## Why Repomix produces too many tokens on a large codebase

The mechanic is simple, and it's the same one in all three tools: **concatenation.** They walk the file tree, respect `.gitignore`, and emit a single artifact — a directory tree plus the file contents, formatted for an LLM. That artifact is a one-shot snapshot. Its size is a function of the repo's size, not your task's size.

That's fine when the repo is small. It stops being fine for three reasons that compound.

**The token budget is a function of the repo, not the question.** If your task touches three functions across two files, you still pay for the whole tree. Repomix's own [FAQ on large-repo limits](https://repomix.com/guide/faq) is candid: there's no fixed size cap, but very large repos are constrained by memory and "the AI tool's context limits," and the recommended fix is to hand-curate — targeted include patterns, inspecting token-heavy files, `--split-output`. Past a certain size, *you* become the retrieval engine.

**Compression helps, but doesn't change the shape.** Repomix's `--compress` uses tree-sitter to keep signatures and structural elements and drop function bodies; the [README](https://github.com/yamadashy/repomix) says it "Reduces token usage by ~70% while preserving semantic meaning." Attribute that to Repomix — it's the vendor's number — and note the [code-compression docs](https://repomix.com/guide/code-compress) flag the feature as experimental, since it removes implementation details (loop logic, internal variables, the bodies). A 70% reduction on a 2M-token dump is still 600k tokens, and you've thrown away exactly the bodies an agent often needs to read. Third-party research that buckets these tools as ["context packing" reaches the same conclusion](https://rywalker.com/research/code-intelligence-tools): packing "often suffices" for small repos under ~10k files, but "breaks down at scale… even with compression, large monorepos exceed context windows."

**Fitting isn't the same as being read.** Even when the dump *does* fit, a flat wall of text degrades accuracy. The ["lost in the middle"](https://pub.towardsai.net/why-language-models-are-lost-in-the-middle-629b20d86152) effect is well documented: models attend best to the start and end of a long prompt and can drop more than 30% when the relevant fact is buried in the middle. A whole-repo concatenation is the worst case — the one function that matters sits in the middle of thousands of lines the model will skim. This is the same wall you hit from the size side when [your codebase is too large for the context window](/gortex/codebase-too-large-for-context-window/): a bigger window doesn't save a structure-free dump.

And there's a quieter cost in agent loops: the dump re-bills. Drop a packed repo into context and you carry it — and pay for it — on every subsequent turn. That's the core of why [your AI agent wastes most of its tokens finding code](/gortex/ai-agent-token-waste-code-retrieval/).

---

## The packers, fairly: Repomix vs Gitingest vs code2prompt

These are good tools. Each owns a distinct one-liner, and each has a case where it's the right answer. All three are MIT-licensed (as of 2026 — check each repo's `LICENSE` before you rely on it), actively maintained, and ship an MCP server — so this isn't a maturity argument.

| Tool | One-line accent | Genuine strength | The shared limit |
|---|---|---|---|
| **[Repomix](https://github.com/yamadashy/repomix)** | Pack your entire repo into one AI-friendly file | Most complete: XML output tuned for Claude, `--compress` (tree-sitter), Secretlint secret scan, `--token-budget` CI guard, MCP server, hosted web | Whole-repo snapshot; re-bills every turn; you curate scope at scale |
| **[Gitingest](https://github.com/coderamp-labs/gitingest)** | Swap `hub` → `ingest` in any GitHub URL | Lowest friction: a public repo to prompt-ready text with zero install ([hosted app](https://gitingest.com/)) | No compression or summarization at all; everything non-`.gitignore`d goes in |
| **[code2prompt](https://github.com/mufeedvh/code2prompt)** | Convert a codebase into one prompt, your way | Fastest (Rust), most customizable: Handlebars templates, TUI, git diff/log scoping, Python SDK, MCP server | Still concatenation; reduction only via glob filtering, no structural compression |

When is a packer the right tool? Genuinely often: a repo the model has never indexed (one-shot, no setup — pack and paste); a small repo where the whole thing fits; sharing context in a ticket as one self-contained artifact; a git-diff-scoped prompt like "review this PR" (code2prompt's diff integration is built for it).

The packers lose only where their core assumption breaks: when the repo is large *and* the session is long, so re-packing the whole repo every turn is the real cost. That's the boundary. Below it, pack. Above it, query.

---

## The Repomix alternative for AI agents: index once, query many

The alternative isn't a better packer. It's a different verb. Instead of concatenating the repo into a prompt, you index it once into a **code knowledge graph** — symbols as nodes, with deterministic `calls`, `implements`, and `references` edges between them — and then *query* that graph for exactly the slice a task needs. (If the graph idea is new, the pillar explains [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/).)

The difference in one table:

| | Packers (Repomix / Gitingest / code2prompt) | Code graph (query) |
|---|---|---|
| **Unit of context** | The whole repo (or a glob-filtered slice) | The task's working set — target symbols + their real dependencies |
| **Cost driver** | Size of the repo | Size of the task |
| **Structure** | Flat text + a directory tree | Traversable call / reference / implements edges |
| **Across a session** | Re-pack / re-bill each turn | Index once; query cheap slices repeatedly |
| **Staleness** | Snapshot at pack time | Live graph, incremental updates |
| **Best at** | Small / one-shot / never-indexed | Large repos, long agent sessions |

This is the same distinction that separates whole-repo embedding indexes from structural ones, covered in [how Cursor indexes your codebase — embeddings vs a structural graph](/gortex/how-cursor-indexes-codebase-embeddings-vs-graph/). The packer's snapshot is "give the model everything"; the graph is "give the model the right thing."

---

## How Gortex queries the graph instead of concatenating

[Gortex](https://github.com/zzet/gortex) is a code graph engine: it indexes a repo into an in-memory knowledge graph and exposes it over a CLI, an MCP server, and a web UI. It's Apache 2.0 (free for commercial use), runs locally with no API key, and covers 257 languages. The relevant part here is the set of tools that replace "pack the whole repo."

**`smart_context` — the task-aware working set.** You describe the task; it returns the minimal set of symbols you need, file-clustered, plus a `blast_radius` block of what a change would touch — in one call. Gortex documents this as replacing **5–10 separate exploration reads.** It's the inverse of concatenation: not "here's everything, find the relevant part," but "here's the relevant part."

```bash
# Don't pack the repo. Ask the graph for the working set.
gortex context --task "add a retry to the webhook dispatcher" --estimate
```

**`context_closure` — the dependency closure under a token budget.** This assembles the *transitive* closure for a target — the symbol, what it calls, what those call — and packs it under a single token budget. It's the honest version of "give me enough context to change this safely" without dumping the tree. A packer can't follow `handler → service → repo` on demand; the graph traverses real edges to do exactly that.

**`export_context` — a portable, graph-selected briefing.** When you *do* want a single artifact to hand off — the packer's actual strength — Gortex can produce one. The difference is that it's graph-selected (the working set), not the whole tree, so it stays small enough to be read rather than skimmed.

**`get_symbol_source` — read a function, not its file.** Underneath all of this are symbol-level reads: fetch one function instead of the 500-line file around it. Gortex reports **~80% fewer tokens** per read versus reading the whole file, and returns a `tokens_saved` count per call.

The aggregate effect shows up in the retrieval benchmark. On the same queries, ripgrep-plus-full-read versus Gortex: `AddObservation` goes 31,530 → **972** tokens, `IsSymbolQuery` 23,027 → **577**, with **median recall@2k of 1.00 for Gortex versus 0.00 for ripgrep**. Headline range: **3–50× fewer tokens per response** (the ~50× peak is on identifier lookups, not every query — don't expect a flat 50×). Recall matters as much as token count: a smaller context that *contains the right code* beats a bigger one that buries it.

Two structural advantages beyond tokens. The edges are **deterministic and verifiable** — `find_usages` returns every real reference with zero false positives, where a text dump leaves the model to pattern-match call sites itself. And the graph is **live**: an fsnotify watch applies surgical per-file patches in ~200ms instead of a 3–5s full re-index, so a long session's context never goes stale the way a pack-time snapshot does. Re-packing the whole repo on turn forty is wasted work; the graph already has turn forty's answer indexed.

### Honest limits

The graph wins on scale and session length — not everywhere, and it costs something the packers don't.

- **Bootstrap is heavier.** Gortex is a binary plus a daemon. A packer is one command (or, for Gitingest, a URL edit). For a single one-shot paste of a small public repo, the graph is overkill — a packer is the right call.
- **No in-browser story.** Gitingest's URL swap works from any browser with zero install. Gortex doesn't.
- **Some refactors are Go-only for now** (`move_symbol`, `inline_symbol`). Search, context, and impact tools are cross-language across the 257-language set.
- **Default embeddings are lightweight.** Semantic search is hybrid (BM25 + vector + RRF) and default-on with embedded GloVe-50d — zero setup, no API key. For stronger semantic recall you opt into MiniLM or OpenAI embeddings. Gortex's edge is the *deterministic graph*, not semantic supremacy.

---

## Which one should you use?

Both approaches are legitimate. A short guide:

| Your situation | Use |
|---|---|
| Small repo, one-shot paste into a chat | **Repomix** or **Gitingest** |
| Public repo, zero install, quick look | **Gitingest** (URL swap) |
| Repeatable local/team prompt, want compression + secret scan | **Repomix** |
| PR review / git-diff-scoped prompt, custom template | **code2prompt** |
| Agent working a large repo across a long session | **A code graph** (Gortex) |
| Model has never seen the repo and you won't index it | **A packer** |

The packers and the graph aren't mutually exclusive, either: pack a small dependency the agent has never indexed, and query the graph for the large codebase the agent lives in.

---

## FAQ

### What is the best Repomix alternative for AI agents?

It depends on session length. For a one-shot paste of a small repo the model has never seen, Repomix and Gitingest are excellent. For an agent working a large repo across many turns, a code-graph engine like Gortex fits better: it indexes the repo once and serves a task-aware working set (`smart_context` / `context_closure`) instead of re-packing the whole repo every turn — 3–50× fewer tokens per response, with deterministic call and reference edges rather than a flat text dump.

### Repomix vs Gitingest — which should I use?

Gitingest is the most frictionless: swap `hub` for `ingest` in a GitHub URL and get raw concatenated text plus a directory tree, no install. Repomix is the more capable local tool — XML output tuned for Claude, tree-sitter `--compress` (Repomix claims ~70% reduction, flagged experimental), Secretlint secret scanning, token-budget CI guards, and an MCP server. Both are MIT-licensed as of 2026. Use Gitingest for a quick paste of a small public repo; use Repomix for repeatable local or team workflows.

### Why does Repomix produce too many tokens on a large codebase?

Because it packs the whole repo into one file, so the output scales with the repo, not your task. Repomix's own FAQ notes very large repos are constrained by memory and the AI tool's context limits, and recommends include patterns and splitting — meaning you hand-curate scope. Even with `--compress`, large monorepos can exceed a 1M-token window. The alternative is to stop packing: index the repo into a graph and fetch only the dependency closure the task needs.

### Is there a code2prompt alternative that queries the codebase instead of concatenating?

Yes. code2prompt, like Repomix and Gitingest, concatenates files into one prompt. A graph engine such as Gortex flips the model: it builds a queryable knowledge graph and answers "what calls this," "what breaks if I change this," and "give me the minimal working set for this task," returning only the relevant code instead of the entire tree.

### How do I turn a GitHub repo into an LLM prompt without blowing the context window?

For small repos, pack it (Repomix, Gitingest, or code2prompt) and optionally compress. For anything past a small repo, don't put the whole thing in the prompt — index it and retrieve a working set. Gortex's `context_closure` assembles the transitive dependency closure under a single token budget, and `export_context` produces a portable, graph-selected briefing instead of a flat dump the model would partly ignore.

---

## Bottom line

Packers and graphs solve different problems. Repomix, Gitingest, and code2prompt turn a repo into one prompt — the right move for a small, one-shot, never-indexed repo, and they do it well. But concatenation scales with the repo, re-bills every turn, and produces a flat wall of text the model reads unevenly. Past a small repo, in a long session, stop packing: index once and query a graph for the working set — 3–50× fewer tokens per response, deterministic call/reference edges instead of a guessed dump, and a live index that doesn't go stale mid-session.

Pack when the repo is small and unseen. Query the graph when the repo is large and the session is long.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
