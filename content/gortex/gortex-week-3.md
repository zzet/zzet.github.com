---
title: "Gortex: Week 3 — Temporal intelligence, PR review, overlay sessions, daemon-first, and 100+ tools"
description: "Ten days of shipping since v0.39.0: Temporal workflow graph (Go + Java cross-language), gortex review PR system, live overlay sessions with speculative execution, daemon-first architecture, LSP Java via jdtls, index freshness provenance, LLM provider registry, and the tool surface crossing 100."
date: 2026-06-15
lastmod: 2026-06-15
draft: false
slug: "gortex-week-3"
keywords: ["gortex", "mcp", "code intelligence", "temporal workflow mcp", "pr review mcp", "overlay sessions mcp", "speculative execution mcp", "gortex daemon", "jdtls mcp", "lsp java mcp", "100 mcp tools", "change_contract mcp", "index freshness mcp", "llm provider registry", "git worktrees mcp"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

Ten days of shipping since [v0.39.0](/gortex/gortex-week-2/), ~200 commits. The tool surface crossed 100. Here's what shipped.

## Daemon-first architecture

`gortex server` — the standalone HTTP server that used to live as its own binary — is gone. The HTTP API is now `gortex daemon --http`, served from the same process that holds the live graph. This simplifies the deployment model considerably: one daemon process, one graph, one set of watchers. `gortex mcp` auto-detects the running daemon and hands off to it; the graph is already warm on first tool call.

Daemon federation landed alongside this. A `gortex proxy` roster lets a single daemon route across local and remote Gortex instances — one daemon manages local repos over a Unix socket, another handles a shared cloud index over HTTPS, and the active daemon picks the right target per query. `gortex daemon server add/remove` manages the roster.

The MCP surface picks this up through a `gortex://workspace` resource that reports the active daemon's bind mode and its entire member set. Agents can query which repos are available without any prior indexing setup.

## Temporal workflow intelligence

The graph now understands Temporal. Workflow definitions, activity registrations, workflow spawns, activity dispatches, signal sends, and query calls are extracted as first-class graph nodes and edges — in both Go and Java, cross-language.

What that means in practice: if a Go orchestration workflow calls a Java activity, the call edge exists in the graph. `flow_between` traces data from the workflow trigger through to the activity result without reading intermediate files. `analyze kind=temporal_orphans` finds activity definitions that nothing dispatches — registered but never called. `analyze kind=temporal_verify` checks that every workflow-spawned activity has a matching registration, and that every signal the workflow sends has a matching signal handler somewhere.

The Java side handles canonical class names and wrapper dispatch — common Temporal Java patterns where the stub is obtained through a typed factory. Both Go and Java Temporal tests get test-classification edges so coverage analysis doesn't conflate Temporal test harness code with production workflow code.

## PR review: `gortex review`

A complete PR review surface, 12 new tools. The approach is graph-grounded: the changeset maps to symbols, symbols carry blast radius and caller fan-in scores, and the graph drops false positives before any LLM sees the diff.

The core tools:

**`review_pack`** — single-call PR review entrypoint. Folds graph-grounded review, per-symbol semantic classification, per-file risk, contract-impact and architecture-boundary checks, and impacted test targets into one response with a BLOCK/REVIEW/APPROVE verdict and a `verification_command`.

**`critique_review`** — adversarial second pass over a prior review's findings. Asks which findings are genuine vs. false positives. Returns kept findings, dropped findings with reasons, and a revised verdict.

**`triage_prs`** — rank open PRs by graph-derived review priority. Five risk axes (blast-radius flow, caller fan-in, coverage gap, security keywords, community span) into one composite score. `use_llm: true` adds an LLM re-rank pass with per-PR rationale.

**`conflicts_prs`** — merge-order conflict risk. Maps each open PR to the graph communities it touches; reports PRs that collide on the same community with a suggested safe merge order. Useful for planning a merge train before a release.

**`suggest_reviewers`** — blends CODEOWNERS matches, recent authorship of changed symbols, and co-change experts into a ranked reviewer list with per-reviewer reasons.

**`post_review`** — posts findings as inline comments on a GitHub PR, anchored to file + line. Secret-redacted before any payload is sent. `dry_run: true` returns what would be posted.

`gortex prs` is the CLI dashboard — lists open PRs with review state (DRAFT/BASE_MISMATCH/CHANGES_REQUESTED/APPROVED/STALE/READY), CI rollup, and merge blockers. `gortex review` runs a full review from the terminal. A GitHub Action template ships for integrating into CI.

## change_contract pipeline

`change_contract` now runs a verdict pass before any write: a risk gate that blocks if blast radius exceeds a configurable threshold, and an ack-TTL ledger that records which callers have been explicitly reviewed. Writes that touch a high-blast-radius symbol without an ack in the ledger are refused. `propagate-delete` drives a fixed-point orphan pass — when a symbol is deleted, the pass checks whether its callers are now dead and propagates the delete upward.

`symbols_for_ranges` maps line ranges from a diff to symbol IDs, which is the bridge between a human-authored diff and the graph's symbol-level safety checks. The edit flow can now go: diff → `symbols_for_ranges` → `verify_change` → ack → `edit_symbol`, with the ledger tracking each step.

`edit_file` and `edit_symbol` both gained a parse-error guard: the file is parsed after write, and if the parse produces errors that weren't present before, the write is rolled back.

Architecture boundaries from community detection feed into the guard rules: moving a symbol across a community boundary trips a guard even if no explicit `.gortex.yaml` rule covers it.

## Live editor buffers and speculative execution

Overlay sessions landed. Editor extensions push in-flight (unsaved) buffers as overlays; Gortex composes a per-request shadow view on top of the immutable base graph and routes every tool call through it. `find_usages`, `get_call_chain`, `get_dependents`, and the rest all see the editor-buffer state without any per-tool changes. The base graph is never mutated.

Built on the same shadow-graph substrate, `preview_edit` and `simulate_chain` answer "what would change if I applied this edit?" without touching disk. Input is a standard LSP `WorkspaceEdit`. Output: touched files, added/removed/renamed symbols, broken callers, broken interface implementors, blast-radius rollup, suggested test targets, and (when an LSP is configured) round-trip diagnostics.

`simulate_chain` runs an ordered sequence of edits with per-step impact and a cumulative rollup. `stop_on_error: true` aborts on the first new ERROR-severity diagnostic. `keep: true` promotes the final simulated state into a real overlay session — so an agent can plan a refactor, simulate it, check nothing breaks, then promote the plan to an editable working state.

Overlay sessions also support branching: N parallel speculative branches off one baseline, so an agent can hold strategy A and strategy B simultaneously and compare them with `compare_branches`.

## LSP Java via jdtls

Java language server (jdtls) integration is fully wired. Hover result parsing is fixed for jdtls's non-standard extension format. Concurrency is bounded so cold starts with large Java projects don't saturate file descriptors. Reconnect logic is hardened for jdtls's longer startup time. Maven and Gradle projects require an explicit trust opt-in (`.gortex.yaml: lsp.java.trust: true`) since they execute build scripts on initialization. Windows URI normalization is fixed — jdtls sends `file:///C:/...` URIs that need a different path roundtrip than POSIX paths.

The diagnostics tools (`get_diagnostics`, `subscribe_diagnostics`, `fix_all_in_file`) work against Java now, including `server-driven capability registration` — jdtls announces features after `initialize`, and Gortex now honours those late registrations instead of silently returning empty results.

## Index freshness provenance

Every indexed file now carries a provenance record: which extractor version parsed it, when it was last indexed, and what the file's content hash was. The extractor-version is salted into cache keys, so an upgrade that changes how a language is parsed automatically invalidates the stale cache entries for that language.

File-read tools (`get_symbol_source`, `get_file_summary`, `get_editing_context`) carry an inline freshness rider on their responses — a small block that reports whether the returned content is current, the last-indexed timestamp, and whether a re-index is pending. Agents can check this before making safety-critical edits.

Background auto-index handles the slow cases: WSL2 mounts and network filesystems that don't send reliable fsnotify events get a periodic reconcile pass keyed off a notify-file written by the watcher. The reconcile trigger is also exposed as `reindex_repository` with a `paths` subset for scoped re-index — useful after a large `git pull` that touches a specific subtree.

## LLM provider registry

Gortex's internal LLM use (for `ask`, `critique_review`, `triage_prs`, and the `use_llm` folds across tools) is now configurable through a provider registry. New providers: Azure OpenAI, GitHub Copilot, Cursor, OpenCode CLI, and custom OpenAI-compatible endpoints. Anthropic's API gets prompt caching (on by default for qualifying requests) and extended thinking (opt-in per call). Claude model sentinels let you pin `claude-sonnet` or `claude-opus` without tracking the full model string; reasoning-effort control (`low`/`medium`/`high`) maps to Anthropic's budget_tokens parameter.

```yaml
# ~/.config/gortex/config.yaml
llm:
  provider: anthropic
  model: claude-sonnet
  reasoning_effort: medium
  # or:
  provider: custom
  base_url: http://localhost:11434/v1
  model: qwen2.5-coder:32b
```

Custom providers are registered through `gortex llm add <name> --base-url <url> --model <model>` and persist to the config file.

## Retrieval: delta packing and RWR

`smart_context` gained `delta_from`: pass a prior context pack's handle and the response includes only the symbols that changed or are new relative to that pack. Useful in agentic loops where the codebase is stable and only the working set shifts — the follow-up pack is a fraction of the original.

Reranking now wires in Random Walk with Restart (RWR) / Personalized PageRank centrality. Symbols that are central to the graph communities (high centrality under RWR) score higher when they're also retrieval-relevant, which pulls in important hubs that exact-match and BM25 alone tend to under-rank.

The eval harness (`gortex eval recall`) gained P@K / R@K / MRR metrics and pluggable pack strategies, so the ranking pipeline can be compared with and without each component. Implicit feedback is Phase 2: per-session symbol picks feed a cluster-scoped frequency signal; forced injection seeds under-explored clusters; negative signals from unused symbols suppress over-ranked noise.

An append-only query log captures zero-result searches for later mining. Zero-result queries are the clearest signal that the retrieval pipeline is missing something the user needs.

## Git worktrees

Git worktrees are now tracked as independent repo instances. Each worktree gets its own namespace in the graph — symbol IDs include the worktree prefix — so two branches can be indexed simultaneously without colliding. The daemon's warm-restart reconcile was updated to handle worktree instance keys correctly on `git worktree` add/remove events.

`gortex track` and `gortex daemon start` both surface worktree instancing through the CLI, and `track_repository` / `untrack_repository` operate on worktree-aware paths. Useful for agents that switch branches frequently and need graph state from both.

## Savings ledger

The savings ledger is now machine-global and persists to a sidecar database. Prior versions accumulated `tokens_saved` in memory per daemon session; a restart zeroed it. Now the ledger survives restarts, accumulates truthfully (only counting saves that actually happened, not estimates), and age-sweeps entries older than a configurable window.

`gortex savings` shows the accumulated total and a per-tool breakdown. `graph_stats` still returns the session-level view; the sidecar backs a longer-horizon picture of where the graph is saving the most context.

---

Two additions worth noting that didn't fit neatly elsewhere:

**EOL-tolerant edits.** `edit_file`, `edit_symbol`, and `batch_edit` now match `old_string`/`old_source` across line-ending differences. An LF-authored replacement matches a CRLF file and the write adopts the file's own endings. `eol_normalized: true` rides on the response. The prior behavior (silent non-match on CRLF repositories, common on Windows) caused a lot of unnecessary Read-before-Edit round-trips.

**Gemini CLI and Antigravity hooks.** `gortex init` now writes lifecycle hooks for Gemini CLI and Antigravity in addition to the existing nine agents. `generate_skill` is graph-aware — the SKILL.md files it produces reflect the actual community structure instead of a static template. The skill-render drift fence checks generated skills against the live graph and flags files that have drifted.

---

Source: [github.com/zzet/gortex](https://github.com/zzet/gortex)

```bash
curl -fsSL https://get.gortex.dev | sh
gortex install
gortex daemon start --detach
gortex init
```
