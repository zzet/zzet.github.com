---
title: "Impact analysis: what breaks if I change this function?"
description: "Impact analysis: what breaks if I change this function? The honest blast-radius menu — tests, LSP, grep — and a sub-millisecond pre-edit gate for AI agents."
date: 2026-06-01
lastmod: 2026-06-06
draft: false
slug: "impact-analysis-what-breaks-if-i-change-this"
keywords: ["impact analysis what breaks if I change this function", "blast radius of a code change", "change impact analysis before refactoring", "how to stop AI breaking your codebase", "verify change before refactor", "blast radius mcp", "pre-edit safety for AI agents"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You ask your agent to change a function signature — add a parameter, tighten a return type, rename a field. It edits the function, edits the two callers it can see, declares victory. Then CI goes red: a caller three modules away, an interface implementor in another package, and a path that only runs in production all relied on the old shape. The agent never looked, because nothing told it to look *before* it wrote.

That is the question this post is about — **impact analysis: what breaks if I change this function?** — and most tooling answers it *after the fact*. The tests catch it where coverage exists. The PR reviewer flags it after the diff is written. The dynamic-dispatch path fails in prod, where it's most expensive. What's missing is **blast-radius awareness before the edit**: a cheap, deterministic check that tells you — and the agent — the reach of a change before a single byte hits disk.

> **Short answer:** to know what breaks before you edit, run impact analysis first — not the tests after. An IDE's call hierarchy shows direct callers one level at a time, in one language, and doesn't list interface implementors a signature change would break. A resolved code graph answers the whole question at once: Gortex's `explain_change_impact` returns a risk-tiered blast radius (callers, interface implementors, affected processes) in sub-millisecond time, `verify_change` checks a proposed new signature against *every* caller and implementor, and `simulate_chain` speculates the edit against the graph without touching disk. It's cheap enough — p95 0.01ms — to gate every agent edit, not just the scary ones.

---

## Why a change breaks distant code (and why agents make it worse)

A breaking change is a graph problem wearing a text disguise. When you alter a symbol's contract — its signature, return type, or the interface it satisfies — every node with an edge pointing at it is potentially affected. The trouble is those edges are invisible in the file you're editing. You see the function; you don't see the forty things that reach it.

Three classes of break are routinely missed:

- **Distant callers.** Anything past the one or two call sites in the current file. Transitive callers — A calls B calls the thing you changed — are invisible without walking the reverse call graph.
- **Interface implementors.** The silent one. In Go, a type implements an interface [just by matching method signatures — there is no `implements` keyword](https://www.gyata.ai/golang/interfaces-in-go). Change an interface method and every matching implementor is now wrong, with *nothing textual to grep for*. The same blind spot exists for duck typing elsewhere.
- **Dynamic-dispatch and reflection paths.** Calls resolved at runtime — virtual dispatch, reflection, config-driven wiring — leave little or no static trace. These pass CI and fail in production.

AI agents make this sharper because of *how* they edit. An LLM optimizes for plausible code, not the dependency graph — it does find-and-replace-shaped edits and misses call sites across modules. Kiro's write-up on program-analysis-backed refactoring puts the principle well: refactoring is a graph-traversal and constraint-satisfaction problem that demands ["precision over plausibility,"](https://kiro.dev/blog/refactoring-made-right/) which is exactly what an LLM is *not* optimizing for. Their rule of thumb is the cleanest statement of the fix: "if it works when you press F2, it works when the agent does it." The agent needs the same deterministic machinery a human's IDE uses — checked *before* it writes, not after.

That's the deeper version of the problem covered in [why hallucination is a context problem, not a model problem](/gortex/stop-ai-agent-hallucinating-code/): the agent isn't dumb, it's blind. It can't reason about a blast radius it was never shown.

---

## The honest menu: how people answer "what breaks?" today

There's no single tool that owns this, so people stack several. Here's the menu, with what each one actually buys — and where it stops.

| Approach | When it answers | Strength | Where it stops |
|---|---|---|---|
| **Run the tests** | After the edit | Catches real runtime breakage, including some dynamic paths | Only where coverage exists; only paths the suite exercises; slow; post-hoc |
| **LSP / F2 rename** | Before, in the editor | Compiler-precise, deterministic, already installed | One language, one client; one call-hierarchy level per request; no interface-implementor enumeration |
| **Call-hierarchy spelunking** | Before, manually | Accurate per level | You recurse by hand for transitive reach; tedious; single-language |
| **grep / find-and-replace** | Before, blind | Works everywhere, no index | False positives (string matches) and false negatives (interface implementors, aliases, generated refs) |
| **AI PR reviewer** | After the edit, pre-merge | Reasons about impact beyond the diff | Reviews a diff already written; not a pre-edit gate; false-positive tax |

A few deserve their names.

**Run the tests** is the honest baseline, and it's just *late*. Tests catch breakage after the edit exists, only where there's coverage, and they [under-report on dynamic dispatch and reflection](https://arxiv.org/html/2407.07804v1). The point isn't to skip tests; it's that they're the wrong *first* gate.

**LSP call hierarchy** is the IDE-native answer, and within its lane it's excellent: deterministic, free, already running. The [LSP 3.17 spec](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/) gives you `prepareCallHierarchy` plus incoming/outgoing calls, and F2 does a semantic rename. Two limits matter for impact work. First, it [returns one level at a time — the client recurses for transitive reach](https://github.com/neovim/neovim/issues/26817), so "everything that breaks" is a manual tree-walk. Second, and more damaging: call hierarchy on an *interface method* does not enumerate the implementors a signature change would break — the exact silent class above. And it's single-language, single-client: fine for a human in one editor, awkward as a scripted gate.

**codanna** ([github.com/bartolli/codanna](https://github.com/bartolli/codanna)) is the closest same-category peer, and a genuinely good local tool — actively maintained, Apache-2.0, v0.9.22 as of May 2026, with sub-10ms lookups across roughly fifteen languages. It owns the literal phrasing: its `analyze_impact` is documented as ["Full dependency graph — what breaks if I change this?"](https://playbooks.com/skills/nickcrew/claude-cortex/codanna-codebase-intelligence) with graduated guidance ("No impact detected. Likely isolated." up to "20+ symbols affected… break the change into smaller parts") and configurable transitive `find_callers`. It's a real impact tool. What it doesn't give you: explicit *risk tiers*, affected-process awareness, an interface-implementor check tuned for signature changes, disk-free WorkspaceEdit speculation, or a map from changed symbols to the exact tests to run.

**Greptile** ([greptile.com](https://www.greptile.com/)) takes the opposite position on the timeline. It [builds a graph of your repo and runs parallel agents that "assess their impact beyond the diff"](https://www.greptile.com/) on every PR, and it's strong at it: an [independent benchmark of 50 real-world PRs](https://medium.com/@pantoai/coderabbit-vs-greptile-ai-code-review-tools-compared-8b535666f708) (Sentry, Cal.com, Grafana) had it [catching 82% of bugs vs CodeRabbit's 44%](https://www.greptile.com/benchmarks) — at a recall cost of 11 false positives to CodeRabbit's 2. The fair framing: Greptile is a *post-write, pre-merge* reviewer. It reasons about a diff that's already written — a different job from a *pre-edit* gate the agent self-checks before producing the edit at all.

**Sourcegraph / Cody** owns precise, symbol-level find-references over large multi-repo estates — real impact data at enterprise scale. The caveat is access and shape: it's a heavy enterprise platform, and the self-serve [Cody Free/Pro tiers were discontinued in 2025](https://sourcegraph.com/pricing) (verify current terms before quoting any price). Not a drop-in local pre-edit gate.

The shared gap: either the menu answers *after* the edit, or it makes you walk one call-hierarchy level at a time in one language with no interface-implementor coverage. Nobody combines pre-edit, risk-tiered, implementor-aware, disk-free speculation cheap enough to run on *every* edit.

---

## Impact analysis that answers "what breaks if I change this function?" before the edit

[Gortex](https://github.com/zzet/gortex) indexes a repository into an in-memory [code knowledge graph for LLMs](/gortex/code-knowledge-graph-for-llms/) and exposes it over MCP, CLI, and HTTP — Apache 2.0, 257 languages, 100+ MCP tools. Because the graph carries resolved `calls`, `references`, and `implements` edges, "what breaks if I change this?" becomes a graph query instead of a text search or a test run. The implementor case is just an edge: the graph stores the `implements` relationship that Go's implicit interfaces give you nothing textual to grep for.

The tools form a pipeline, escalating from "tell me the reach" to "let me try it without committing."

**1. `explain_change_impact` — the risk-tiered blast radius.** Given a symbol, it returns the reach *graded by risk*, including affected processes, not just symbols. That's the difference from a bare caller count: a leaf utility with two local callers and a widely-implemented interface method are not the same risk, and the tiering says so.

```bash
# CLI blast-radius demo (the agent calls the explain_change_impact MCP tool)
gortex query dependents "Store.Get"
```

**2. `verify_change` — proposed signature vs every caller and implementor.** You hand it the *new* shape, and it checks that shape against all callers and all interface implementors — the step LSP call hierarchy can't do, because it explicitly enumerates the implementors a signature change would break.

```bash
# MCP (what your agent calls)
verify_change(symbol: "Store.Get", new_signature: "Get(ctx, id) (*Row, error)")
```

**3. `get_dependents` — the raw reverse reach.** The blast-radius primitive underneath: everything depending on the symbol. Pair it with [graph-verified refactoring — safe rename, dead-code detection, and cycle checks](/gortex/graph-verified-refactoring/) when the next step is to fix every dependent.

**4. `preview_edit` / `simulate_chain` — speculate without touching disk.** The part the rest of the menu doesn't offer: describe a WorkspaceEdit and ask "what would change if I applied this?" — the graph answers against the whole repo *without writing a byte*. The agent can try the edit, see the cascade, and back out, all in-memory.

**5. `get_test_targets` — changed symbols to the exact tests.** Once you know what changes, it maps the changed symbols to the test files and run commands that cover them, cross-repo aware. This is where pre-edit impact analysis hands back to tests deliberately: see the blast radius first, then run *exactly* the tests that matter.

### The economics that make it gate-able

The reason this runs on every edit, not just the frightening ones, is cost. Gortex precomputes a depth-3 reach index, so an impact query is O(seeds × reach) map lookups rather than a fresh graph walk. Measured impact analysis lands at **p95 0.01ms — 100× under a 1ms budget** (gortex repos, ~71k nodes, M3 Max; a single-machine warm-daemon number, absolute timings vary 2–5× by hardware). When asking "what breaks?" costs effectively nothing, you stop rationing the question.

### Wire it into a pre-edit hook so the agent self-checks

The payoff for AI safety is automation. Claude Code's [PreToolUse hooks fire before any tool the agent calls](https://code.claude.com/docs/en/hooks-guide) — including MCP tools — and see the tool name, arguments, and working directory, which makes them the chokepoint to *gate an edit before it happens*. Run `verify_change` in a PreToolUse hook and block the write when callers or implementors would break, and the agent self-checks every edit. That's the concrete answer to **how to stop AI breaking your codebase**: not a smarter model, a deterministic gate the agent can't route around.

| | grep / find-replace | LSP call hierarchy | AI PR reviewer | Gortex `explain_change_impact` / `verify_change` |
|---|---|---|---|---|
| When | before, blind | before, in editor | after the diff | **before, in graph** |
| Transitive reach | no | one level/request | yes | **precomputed depth-3** |
| Interface implementors | no | **no** | sometimes | **yes (`implements` edge)** |
| Risk tiers / processes | no | no | partial | **yes** |
| Disk-free speculation | no | no | n/a | **`simulate_chain`** |
| Cheap enough per edit | n/a | manual | no | **p95 0.01ms** |

### Determinism vs approximation

One distinction worth holding onto: a code graph's edges — `calls`, `implements`, `references` — are *verifiable*, resolved relationships. Embedding-similarity approaches approximate "related code" by vector distance, which is fine for recall but no guarantee that X actually calls Y. For impact analysis you want the deterministic version: if the graph says these forty things reach your symbol, they reach it. (Gortex's retrieval is hybrid — BM25 + default-on GloVe-50d embeddings + RRF + graph reranking — but the impact edges are graph-resolved, not similarity-guessed.) The same deterministic graph powers [SAST and taint analysis from the code graph](/gortex/sast-taint-analysis-from-code-graph/), where "does tainted data reach this sink?" is the security cousin of "what breaks if I change this?".

### The honest concession: static analysis can't see everything

This is not a test replacement, and saying so is the point. Static analysis cannot fully resolve **dynamic dispatch, reflection, or config-driven wiring**. Dynamic frameworks like [Spring and Rails produce incomplete call graphs](https://www.in-com.com/blog/advanced-call-graph-construction-in-languages-with-dynamic-dispatch/), because real tools eschew prohibitively conservative reflection handling, and [static analysis does not record the actual targets of reflective calls](https://arxiv.org/html/2407.07804v1). So a graph can *under-report* on those paths. Position it correctly: the cheap, deterministic **first** gate that catches the obvious breakage — distant callers, interface implementors, transitive reach — before you write, with tests still owning the dynamic paths after. Use both, not either/or. (Two more limits, for fairness: `move_symbol` and `inline_symbol` are Go-only for now, and the headline number here is the sub-millisecond impact stat, not the token figure.)

---

## FAQ

### How do I know what breaks if I change a function before I edit it?

Run impact analysis first, instead of relying on the tests after. An IDE's call hierarchy shows direct callers one level at a time, in one language, and you recurse by hand. A code-graph tool answers the whole question at once: Gortex's `explain_change_impact` returns the full risk-tiered reach — including interface implementors and affected processes — in sub-millisecond time, and `verify_change` checks a proposed new signature against every caller and implementor before you touch disk.

### Why do AI coding agents keep breaking distant callers when they refactor?

Because an LLM optimizes for plausible code, not the dependency graph. It does find-and-replace-shaped edits and misses call sites across modules, dynamic imports, and interface implementors — Kiro calls it "precision over plausibility." The fix is to give the agent blast-radius awareness before it writes: a PreToolUse hook that runs `verify_change` and blocks the edit when callers or implementors would break.

### Isn't running the tests enough to catch a breaking change?

Tests catch breakage after the edit, only where coverage exists, and only for the paths the suite exercises — dynamic dispatch and reflection often slip through. Pre-edit impact analysis is the cheaper first gate: see the blast radius before writing, then run exactly the tests that cover the changed symbols (`get_test_targets`). Use both, not either/or.

### How is this different from LSP call hierarchy or grep?

Grep gives false positives (string matches) and false negatives (misses interface implementors, aliases, and generated references). LSP call hierarchy is accurate but single-language, returns one level per request, and doesn't enumerate the interface implementors a signature change would break. A code graph gives deterministic `calls` / `implements` / `references` edges, transitive reach in one call, and is cheap enough — p95 0.01ms — to gate every edit.

### Can static impact analysis catch everything?

No. Static analysis can't fully resolve dynamic dispatch, reflection, or config-driven wiring (Spring, Rails), so it can under-report there. Use it as a fast, deterministic first gate for the obvious breakage — distant callers, interface implementors, transitive reach — and keep tests for the dynamic paths.

---

## Bottom line

"What breaks if I change this function?" is a graph question, and most tooling answers it at the wrong time — the tests after the edit, the PR reviewer after the diff, the prod failure after deploy. The pre-edit options each fall short: grep is blind to implementors and noisy with false positives; LSP call hierarchy is precise but single-language, one level at a time, and silent on interface implementors. A resolved code graph closes the gap: `explain_change_impact` for the risk-tiered blast radius, `verify_change` for a proposed signature against every caller and implementor, `simulate_chain` to try the edit without touching disk, and `get_test_targets` to run exactly the right tests after. At p95 0.01ms it's cheap enough to gate every agent edit through a PreToolUse hook — the deterministic first line on how to stop AI breaking your codebase, with tests still owning the dynamic paths static analysis can't see.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
