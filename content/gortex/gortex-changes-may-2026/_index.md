---
title: "What's new in Gortex — May–June 2026"
date: 2026-06-06
draft: false
description: "A month of code-intelligence upgrades across Gortex v0.19 to v0.39: smarter retrieval, wider language ingest, deeper analysis, better agent ergonomics, and a rebuilt storage layer."
summary: "A month of code-intelligence upgrades across Gortex v0.19 to v0.39: smarter retrieval, wider language ingest, deeper analysis, better agent ergonomics, and a rebuilt storage layer."
tags: ["Gortex", "Release Notes"]
keywords: ["Gortex", "code intelligence engine", "code graph", "MCP server", "Model Context Protocol", "AI coding agents", "release notes", "changelog"]
---

> **New to these changes?** Start with the **[plain-English guide](00-plain-english-guide)** — what each update means and exactly what to do about it, no graph theory required. The deep dives below go a level deeper.

## It's been a busy month

We went quiet on the blog for a few weeks — not because nothing happened, but because *a lot* did. Since our last post (Gortex v0.19, early May) we've shipped twenty releases and around eight hundred commits. This post is the catch-up: the themes that matter, grouped so you can skim to what you care about.

If there's one headline, it's this: **code retrieval got measurably smarter.** Gortex's whole reason for existing is to hand an agent — or you — the *right* code with the *fewest* tokens, and this month we turned that from something we tuned by feel into something we *measure*. Graph-centrality reranking, per-query lexical/semantic blending, query understanding, and a real evaluation harness (P@K / R@K / MRR) all landed together. That's section 1, and it's the one to read first.

Around that headline, the graph learned a lot more languages and frameworks, the storage layer was rebuilt around a pure-Go default backend, and the agent-facing MCP surface grew a set of features that make Gortex a much better teammate for coding agents.

> **TL;DR**
> - **★ Smarter retrieval (the headline)** — graph-centrality reranking, per-query BM25/semantic blending, a thesaurus expansion layer, zero-result query rescue, and a real eval harness (P@K / R@K / MRR). Retrieval quality is now measured, not guessed.
> - **Wider ingest** — Terraform, Helm, Ansible, .NET, COBOL/JCL, Luau, Quarto, C/C++ macros, MCP server configs, **plus images and PDFs as first-class graph nodes**.
> - **Deeper resolution** — a framework dynamic-dispatch synthesizer engine and cross-language bridges (React Native, Swift↔ObjC, Expo, Spring/Symfony DI, MyBatis, WebSocket/SSE).
> - **New analysis** — interprocedural complexity & bottleneck scoring, and first-class capability edges (`reads_env`, `executes_process`, `accesses_field`).
> - **Better agent ergonomics** — session memory, cross-session development memories, a multi-agent coordination registry, safer edit tools, and graph-aware LLM model routing.
> - **A rebuilt storage layer** — a pure-Go SQLite backend (now the default, no CGO needed for the store), warm-restart reconcile, and unified `~/.gortex` home.

---

## 1. Retrieval got a lot smarter

**→ Read the deep dive: [Retrieval got a lot smarter](01-retrieval-got-smarter)**

This was the biggest single investment of the month. Gortex's job is to hand an agent (or you) the *right* code with the *fewest* tokens, and we pushed hard on both halves.

- **Graph-centrality reranking.** We wired Random-Walk-with-Restart / Personalized PageRank centrality into the rerank pipeline, so results that sit at the structural heart of the relevant subgraph float to the top — not just the ones that match keywords. To keep it fast, the walk is cached and keyed by a per-package Merkle hash, so it recomputes incrementally instead of from scratch.
- **Per-query blend tuning.** The BM25 ↔ semantic mix is no longer a fixed knob; each query gets a continuously tuned `alpha`, so identifier lookups lean lexical and natural-language questions lean semantic.
- **Query understanding.** A concept-relatedness thesaurus layer expands queries by meaning; "keyword-soup" queries (no operators, just terms) are detected and split; and zero-result identifier queries are *rescued* by decomposing them into leaf terms instead of returning nothing.
- **Docs as a first-class channel.** Documentation now has its own retrieval channel with prose-tuned reranking, so a question about *how something works* surfaces the doc that explains it, not just the symbol that implements it.
- **Better embeddings.** Launch-time embedding-model variant selection, plus AST-aware sub-chunking for Ruby, PHP, Kotlin, and Swift, and an exact-cosine recovery pass after reranking.
- **Generated-file down-ranking.** When a generated file shadows a real implementation, the real one wins.
- **A real eval harness.** We added a `gortex eval pack` harness that scores retrieval with P@K / R@K / MRR (and LLM format-comprehension), plus an append-only query log with zero-result mining and a second iteration of implicit-feedback learning. Retrieval quality is now something we *measure*, not just tune by feel.

---

## 2. smart_context: less, but more relevant

**→ Read the deep dive: [smart_context: less, but more relevant](02-smart-context)**

`smart_context` is the one-call "give me the working set for this task" tool.  This month it got smarter about *what* to include and *how densely*:

- **Delta packing** — pass `delta_from` to get only what changed since a previous context pack, instead of re-sending the world.
- **Dependency-closure selection** — a new context-closure pass pulls in the symbols a change actually depends on.
- **Budget that scales with the project** — seed count and token budget now scale to graph size, and the working set is clustered by file before packing.
- **Fidelity tiers** — per-glob `full` / `compress` / `omit` controls, plus skeletonization of large interchangeable symbol families, so you spend tokens where they matter. (`compress_bodies` elides function bodies to signatures + doc-comments across 14 languages — roughly 30–40% of the original tokens.)

---

## 3. Cross-language resolution & framework awareness

**→ Read the deep dive: [Cross-language resolution & framework awareness](03-cross-language-resolution)**

A graph is only as good as its edges. We closed a large cluster of resolution gaps, anchored by a new **framework dynamic-dispatch synthesizer engine** that recovers edges frameworks create at runtime:

- **Cross-language bridges** — React Native (JS ↔ native), Swift ↔ Objective-C, Expo Modules, and Fabric/Codegen view components.
- **DI containers** — Spring and Symfony interface-to-implementation bindings.
- **Data access** — MyBatis mapper XML linked to DAO methods and SQL; cross-language call sites linked to SQL functions.
- **Transport** — WebSocket upgrade and SSE stream edges; HTTP contracts now join router mount prefixes into full paths and understand client-wrapper aliases.
- **Language depth** — Rust impl-block / self-receiver / module-path resolution, Kotlin companion-object dispatch, C# interface-vs-base-class discrimination, Svelte re-exports and `package.json` exports subpaths.
- **External calls qualified by default**, plus per-reference context labels and a `find_usages` context filter so you can ask "where is this used *as a test*" vs. "in production."
- **A "why unresolved" taxonomy** — `analyze kind=resolution_outcomes` now gives a structured account of *why* an edge couldn't be resolved, instead of a silent gap.

---

## 4. Many more languages — and file types

**→ Read the deep dive: [Many more languages — and file types](04-languages-and-file-types)**

Gortex indexes a lot more of your repo than it used to:

- **New extractors:** Terraform/HCL (cross-block references + addressable blocks), Helm (named templates, `include`/`template` calls, chart dependencies), Ansible (playbooks/roles/tasks/handlers + module-call edges), .NET (`.sln`/`.csproj` project graph + `ProjectReference`/`PackageReference`), Quarto `.qmd`, Luau, COBOL paragraphs + JCL job streams, C/C++ preprocessor macros (`KindMacro` + hidden-call recovery), and MCP server configs.
- **Multimodal ingest** — image files become graph nodes (format, dimensions, sha256) and PDFs become per-page searchable document nodes. Your specs and diagrams are part of the graph now.
- **Live database schemas** — `gortex db schema --postgres <dsn>` ingests a running database's tables and columns as graph nodes.
- **Extensibility without a fork** — two new SPIs let you teach Gortex new tricks from config: a declarative **fallback-chunker** for grammar-less languages (`index.fallback_chunkers`) and a subprocess **extractor-plugin** for custom post-parse passes (`index.extractor_plugins`).

---

## 5. Deeper analysis

**→ Read the deep dive: [Deeper analysis](05-deeper-analysis)**

The `analyze` dispatcher kept growing (it's now a 60-kind surface). Highlights from this window:

- **Interprocedural complexity & bottlenecks** — cognitive complexity, loop depth, and *transitive* hidden-O(nᵏ) loop nesting across call boundaries, plus unguarded-recursion detection.
- **First-class capability edges** — `reads_env`, `executes_process`, and `accesses_field` are now traversable edges. You can ask "what reads `$AWS_SECRET`," "what shells out," or "what writes this field" in a single hop — the spine of a least-privilege or supply-chain audit.
- **Analyzer parity for Java** — dead-code, entry-point, and process analysis now work as well on Java as on Go.

---

## 6. A much better teammate for coding agents

**→ Read the deep dive: [A much better teammate for coding agents](06-agent-teammate)**

A lot of the month went into making Gortex pleasant and safe for autonomous agents to drive over MCP.

- **Session memory** — `save_note` / `query_notes` / `distill_session`: a per-session scratchpad that survives context compaction, auto-linked to the symbols you mention.
- **Development memories** — `store_memory` / `query_memories` / `surface_memories`: *cross-session*, workspace-wide knowledge (invariants, gotchas, conventions, decisions) that's surfaced proactively when its anchor symbols enter the working set. Knowledge that compounds the longer a team uses Gortex.
- **Multi-agent coordination** — an `agent_registry` tool and `KindAgent` nodes, with `UserPromptSubmit` pre-turn context injection and verified subagent MCP-tool propagation.
- **Safer edits** — `dry_run` and unified-diff previews on the edit tools, a discriminated-union schema for `batch_edit` (heterogeneous ops in one call), and a post-edit syntax-health check with an external-linter bridge (`lint_file`).
- **Smaller niceties** — `pattern` accepted as an alias for `query` on search tools, and an orphan watchdog that closes the MCP proxy when its parent process dies.
- **Overlay sessions** — editor extensions can push unsaved buffers as a per-session shadow graph; this month overlays gained **branching and parallel speculative sessions**.
- **Artifacts** — non-code knowledge files (DB schemas, API specs, ADRs) declared in `.gortex.yaml` are indexed as `artifact` nodes and linked to the code that implements them.

---

## 7. LLM providers & graph-aware routing

**→ Read the deep dive: [LLM providers & graph-aware routing](07-llm-providers-routing)**

Gortex's `ask` agent and assisted search can run on whatever model you already pay for:

- **Graph-aware model routing** — `ask` can route to a cheaper or more capable model based on graph-derived task complexity (off by default; configure `llm.routing`). The chosen model and complexity ride along on the response.
- **Providers** — Anthropic, OpenAI, Ollama, Gemini, AWS Bedrock (SigV4, no AWS SDK), DeepSeek, plus subprocess providers that reuse your existing CLI sign-ins (`claudecli`, `codex`). Point Gortex at whatever you already pay for.

---

## 8. A rebuilt storage & performance layer

**→ Read the deep dive: [A rebuilt storage & performance layer](08-storage-and-performance)**

Less visible, but foundational. We spent serious time on how Gortex persists and reloads the graph:

- **Pure-Go SQLite backend, now the default.** Built on `modernc.org/sqlite`, with FTS5-backed symbol search — the store no longer requires CGO. Choose with `--backend memory|sqlite`.
- **A SQLite sidecar** for session notes, development memories, saved scopes, and notebooks — they now survive daemon restarts cleanly.
- **Warm-restart reconcile** — persisted file mtimes let the daemon come back in seconds instead of re-indexing the world, and we fixed a class of phantom-deletion and reconcile bugs along the way.
- **Indexed graph capabilities** — a large batch of typed Store capabilities pushed expensive global graph walks (dead-code, implements/overrides inference, centrality, fan-in/out) into indexed, backend-native operations. Big wins on large repos.
- **Unified home** — all per-user state now lives under a single `~/.gortex` tree (config, store, cache, models, memories), with automatic migration on first run.
- **Per-repository incremental clone detection** with a maintained LSH index, so edit-time clone detection is O(edited file), not O(repo).

---

## 9. Multi-workspace & git worktrees

**→ Read the deep dive: [Multi-workspace & git worktrees](09-multi-workspace-worktrees)**

Gortex now tracks **git worktrees as independent repo instances** — each worktree gets its own graph, surfaced through the CLI, daemon, and MCP. If you work across several worktrees of the same repo, their graphs no longer collide.

---

## Upgrade notes

- The default graph backend is now pure-Go SQLite — no CGO needed for the store.  (The tree-sitter parsers still use CGO at build time.)
- Per-user state moved to `~/.gortex`; existing config/cache is migrated automatically on first run.

---

*Thanks for reading. Gortex is Apache-2.0 — try it, file issues, and tell us what your agents need next.*

