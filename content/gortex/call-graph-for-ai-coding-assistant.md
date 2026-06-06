---
title: "Call graph for an AI coding assistant: who calls this?"
description: "A call graph for AI coding assistant work: why grep can't trace callers, how code2flow/codanna compare, and resolved cross-language call chains."
date: 2026-05-25
lastmod: 2026-06-06
draft: false
slug: "call-graph-for-ai-coding-assistant"
keywords: ["call graph for AI coding assistant", "get all callers of a function", "trace function call chain across files", "generate call graph from source code", "who calls this function", "cross language call graph"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

A production bug lands. The stack trace ends at a function called `normalizeAmount`, and the obvious next question is the one every debugger asks: *who calls this?* You run `rg normalizeAmount`, get nineteen hits, and now you're reading. Three are the definition and its tests. Two are in comments. One is a string in a log line. Some are real calls; some pass the function by reference; one crosses into a different service over HTTP and grep can't see it at all. You are doing, by hand, the work a call graph for AI coding assistant workflows should do for you — except your agent doesn't have one either, so it's reading the same nineteen lines and guessing.

"Who calls this?" is the question text search structurally cannot answer. grep matches the *string*, not the *call edge*. It can't tell a call from a comment, it can't span files or packages, and it certainly can't follow a request across a language boundary into another repo. For a human skimming, that's annoying. For an agent that acts on what it finds, every wrong guess is a wrong edit or a missed dependency.

> **Short answer:** To get all the callers of a function reliably you need a *resolved call graph*, not a text search. grep/ripgrep match a name — including comments, strings, and the definition — and can't span files, packages, languages, or repos. A code-graph engine answers it directly: a reverse query returns every function that actually calls your target (zero false positives), a forward query traces the chain it triggers, and a blast-radius query shows everything downstream. Gortex exposes all three as MCP tools — `get_callers`, `get_call_chain`, and `get_dependents` — across 257 languages and across repositories.

## Why "who calls this function?" is hard

A [call graph](https://en.wikipedia.org/wiki/Call_graph) is a directed graph whose nodes are functions and whose edges are "A calls B." For a toy program that's mechanical. At real scale it hits two walls no amount of clever regex gets past.

The first wall is text versus structure. grep is a line-oriented scanner — fast and correct at finding lines that contain a pattern. But "lines containing `normalizeAmount`" is not "places that *call* `normalizeAmount`." grep has no notion of scope, type, or which token is the callee. A practitioner write-up, [The Code Question grep Can't Answer](https://medium.com/@wernerk/the-code-question-grep-cant-answer-057bfc8d7fe2), puts it plainly: grep finds every file containing a name in milliseconds but can't distinguish a call from a comment, and structural tools like ast-grep and Semgrep analyse each file independently and build no persistent cross-file graph — so they still can't answer "who calls this?" across a codebase.

The second wall is deeper, and it bounds every tool here, Gortex included. The exact static call graph is a [provably undecidable problem](https://en.wikipedia.org/wiki/Call_graph). Static analysis can only *over-approximate*: it aims to include every real call edge, accepting that it may add some that never fire. Resolving calls precisely needs alias analysis the moment a language has dynamic dispatch (Java, C++), first-class functions (Python, JavaScript), or function pointers (C). A [comparative study of static JavaScript call graphs](https://arxiv.org/html/2405.07206v1) found no single tool hits both high precision and high recall — one reached 99% precision but 91% recall, a third only 49% recall — because edges from `eval()`, `bind()`, and `apply()` aren't visible in source. Reflection makes it worse: [static generators don't model methods invoked via reflection](https://www.in-com.com/blog/advanced-call-graph-construction-in-languages-with-dynamic-dispatch/), and dynamic dispatch hides which implementation a virtual call site reaches.

So the honest framing is not "perfect call graph." It's "a *resolved* call graph that recovers far more than text search, and tells you where it had to give up."

## The solution space, fairly

There's a real menu of approaches here. Each fixes a failure of the one below it and adds a cost.

| Approach | Answers "who calls this?" | Cross-file | Cross-language | Cross-repo | Agent-facing |
|---|---|---|---|---|---|
| grep / ripgrep | no (matches text) | n/a | n/a | no | shell only |
| ast-grep / Semgrep | no (per-file, no graph) | no | per-pattern | no | partial |
| `code2flow` | forward diagram only | within one project | no (one lang/run) | no | no (offline diagram) |
| `codanna` | yes (`find_callers`) | yes | ~13–16 langs | not documented | MCP / CLI |
| resolved code graph | yes (both directions) | yes | yes | yes | MCP / CLI / HTTP |

### code2flow — the diagram generator

[code2flow](https://github.com/scottrogowski/code2flow) (MIT, thousands of GitHub stars) is the best-known "generate call graph from source code" tool, and it's good at what it does: a zero-config, AST-based generator that produces a clean diagram of one project's call structure. If you want to *look at* the shape of a Python or JavaScript module, it's a fine choice.

Its limits are the ones it admits to. It supports only Python, JavaScript, Ruby, and PHP, one language per run, and its README states the rule directly: *"No algorithm can generate a perfect call graph for a dynamic language."* It skips lambdas, functions that are renamed or passed around, undefined functions, and duplicate-named functions. There's no reverse "who calls me" query, no cross-repo, no MCP, no token budget — it's an offline diagram, not something your agent can query mid-task.

### codanna — local MCP code intelligence

[codanna](https://github.com/bartolli/codanna) (Apache-2.0 as of 2026-06-06, hundreds of stars) is the closest direct peer for agents — a local code-intelligence MCP server with real caller tools: `find_callers` (reverse), `get_calls` (forward), `find_symbol`, and semantic doc search, with sub-10ms lookups. This is legitimate local code intelligence with actual call-graph queries, not a diagram. As of 2026-06-06 it covers roughly 13–16 languages (sources vary) within a single indexed repo; it needs a ~150 MB embedding model, and Windows support is experimental. Cross-language and cross-repo call resolution aren't documented — not the same as "absent," but not something the docs claim either.

### Greptile — whole-codebase review, not a caller API

[Greptile](https://www.greptile.com/docs/introduction) is a different shape entirely. On repo connect it builds a complete codebase graph — every function, class, and dependency — and does multi-hop investigation per pull request. It's strong at automated code review with whole-codebase context. But that graph is *internal review infrastructure*, not a developer-facing `get_callers`/`get_call_chain` API you can point your own agent at, and it's a paid SaaS (as of 2026-06-06, base + usage pricing — $30/seat/month including 50 reviews, then $1 per additional review). Excellent for PR review; not the thing you wire into your editor agent to answer "who calls this?"

The wedge none of them quite fills: a *resolved, cross-language, cross-repo* call graph exposed directly to your agent as queryable tools.

## A call graph for AI coding assistant work: how Gortex answers it

[Gortex](https://github.com/zzet/gortex) indexes a repository into an in-memory [code knowledge graph for LLMs](/gortex/code-knowledge-graph-for-llms/) and exposes call relationships over MCP, CLI, and HTTP. The "who calls this?" question actually has three shapes, and Gortex maps each to its own tool.

| Question | Direction | Tool | What you get |
|---|---|---|---|
| Who calls X? | reverse | `get_callers` | every function that actually calls X |
| What does X end up calling? | forward | `get_call_chain` | the chain X triggers, token-budgeted |
| What breaks if X changes? | downstream | `get_dependents` | the blast radius reachable from X |

```bash
# Reverse — who calls this? (debugging a regression)
gortex query callers normalizeAmount

# Forward — what chain does this kick off? (onboarding)
gortex query calls handleCheckout --max-tokens 4000

# Blast radius — what's downstream? (impact analysis before an edit)
gortex query dependents normalizeAmount
```

```jsonc
// What the agent calls over MCP
get_callers({ symbol: "normalizeAmount" })
get_call_chain({ symbol: "handleCheckout", max_tokens: 4000 })
get_dependents({ symbol: "normalizeAmount" })
```

Four properties make this an agent-grade answer rather than "grep but accurate."

**1. Reverse callers with zero false positives.** `get_callers` reads call edges the resolver built, so every result is a function that genuinely calls your target — never a comment, a string, or the definition. On a warm daemon its measured **p95 latency is 0.02ms** (gortex repos, ~71k nodes, M3 Max — a single-machine number, reproducible via `gortex bench`; absolute timings vary 2–5× by hardware). An agent can fan out caller queries across a whole regression surface without spinning up a language server per request — the same resolved-edge foundation behind [finding usages without grep's false positives](/gortex/find-usages-without-grep-false-positives/).

**2. A forward chain that fits the context window.** `get_call_chain` follows execution forward, hop by hop, across file and module boundaries. Critically, it's *token-budgeted* — you cap the chain at a token count so the trace fits the prompt instead of dumping whole files into context. For an agent onboarding to a service, that's the difference between "trace `handleCheckout` to its leaves" and "read forty files and hope."

**3. Cross-language and cross-repo by construction.** This is the real moat. code2flow is four languages, one project; codanna is roughly 13–16 within one repo. Gortex builds the call graph over **257 languages** and links calls that cross service boundaries via canonical contract IDs — HTTP routes, gRPC (proto), GraphQL, message topics (Kafka/RabbitMQ/NATS/Redis), WebSocket — normalized to identifiers like `http::GET::/api/users/{id}`. A TypeScript frontend calling a Go service shows up as a connected edge, not a dead end.

**4. Blast radius in sub-millisecond.** `get_dependents` answers "what's downstream of X?" from a precomputed depth-3 reach index — map lookups, not a graph walk per query. Measured impact **p95 = 0.01ms**, roughly 100× under a 1ms budget, cheap enough to ask on *every* edit. That's the building block for [impact analysis — what breaks if I change this function](/gortex/impact-analysis-what-breaks-if-i-change-this/).

### The framework gap, and the honesty move

Static analysis silently drops the edges modern frameworks create at runtime. Dependency-injection containers wire an interface to an implementation by configuration. RPC and message queues turn a call into a network hop. These [runtime dependencies are invisible to plain static analysis](https://www.augmentcode.com/tools/microservices-impact-analysis). Gortex includes a framework dynamic-dispatch synthesizer that recovers many of them — DI bindings, RPC endpoints, message topics — so they appear as edges instead of vanishing.

Here's the part that matters more than any feature: it doesn't pretend the undecidable problem is solved. When Gortex can't resolve an edge, it surfaces that rather than dropping it. The `analyze` tool with `kind: "resolution_outcomes"` reports the edges it could *not* trace, so an agent knows where the graph is uncertain instead of treating a gap as a confident "no callers."

```jsonc
// What the agent calls over MCP
analyze({ kind: "resolution_outcomes" })
```

That turns the concession into a signal. A missing caller might mean "nothing calls this" or it might mean "called through reflection we couldn't resolve" — and you get told which.

### Three use cases

- **Debug a regression.** Start at the failing function, run `get_callers`, and walk back up the reverse graph — across files and packages — instead of grepping a name and triaging comments.
- **Onboard to an unfamiliar service.** Pick the entry point and run a token-budgeted `get_call_chain` forward. You get the execution shape in one bounded answer rather than opening files until you lose the thread.
- **Impact analysis before an edit.** Run `get_dependents` on the symbol you're about to change. Sub-millisecond blast radius means you check *before* the edit, not after CI goes red.

## Honest limits

No static call graph resolves everything, and Gortex is not the exception. Reflection, `eval`/`bind`/`apply`, and code generation defeat purely static resolution by construction — that's the undecidability result, not an implementation gap. Gortex recovers many *framework* patterns (DI, RPC, message topics) that plain static analysis misses, but it does not claim to recover all dynamic dispatch, and `resolution_outcomes` exists so it can say so. Like any resolved-graph tool it needs a built index and a running daemon — for a one-off literal search in an unindexed directory, grep is still the faster reach, and a [grep replacement for AI agents spanning lexical, structural, and graph search](/gortex/grep-replacement-for-ai-agents/) is the broader story this fits into. The latency figures are single-machine warm-daemon numbers — reproducible relative claims, not hardware guarantees.

## FAQ

### How do I get all the callers of a function across my whole codebase?

Use a resolved call graph, not text search. grep/ripgrep find every file that mentions the name — including comments, strings, and the definition — but can't tell which are actual calls, and can't span files, packages, or repos. A code-graph engine like Gortex answers it directly: `get_callers` returns every function that genuinely calls your target, with zero false positives, around 0.02ms p95 on a warm daemon, across all files, packages, and even other repositories.

### Can I trace a function call chain across multiple files?

Yes. A forward call chain follows execution from a starting function through everything it calls, hop by hop, even across file and module boundaries. Gortex's `get_call_chain` does this and is token-budgeted, so an AI agent can trace the chain without overflowing its context window — unlike dumping whole files into the prompt.

### Why can't grep answer "who calls this function?"

grep matches text, not code structure. It can find every file containing `authenticate` in milliseconds, but it can't distinguish a call from a comment, a string, or the definition, and it has no notion of which functions call which. ast-grep and Semgrep understand syntax within a single file but build no persistent cross-file graph, so they still can't answer "who calls this?" across a codebase. You need a resolved call graph for that.

### Is there a call graph tool that works across multiple programming languages?

Most aren't. code2flow handles only Python, JavaScript, Ruby, and PHP, one language per run; codanna covers roughly 13–16 languages within a single repo. Gortex builds a cross-language, cross-repo call graph over 257 languages and links calls that cross service boundaries (HTTP, gRPC, GraphQL, Kafka, RabbitMQ, NATS, WebSocket) via canonical contract IDs — so a TypeScript frontend calling a Go service shows up as a connected edge.

### Can a static call graph be 100% accurate?

No — the exact static call graph is a provably undecidable problem, so every static tool is an approximation. Dynamic dispatch, first-class functions, function pointers, reflection, and runtime tricks like `eval`/`bind`/`apply` create edges that can't be resolved from source. Gortex recovers many framework-created runtime edges (DI, RPC, message topics) that plain static analysis misses, and reports the ones it couldn't resolve via `resolution_outcomes` instead of silently dropping them.

## Bottom line

"Who calls this?" is the question grep was never built to answer — it matches a string, not a call edge, and stops at the file boundary. The fix is a resolved call graph in three shapes: reverse callers for debugging a regression, a token-budgeted forward chain for onboarding, and a sub-millisecond blast radius for impact analysis. Gortex maps those to `get_callers` (0.02ms p95), `get_call_chain` (budgeted to fit the context window), and `get_dependents` (0.01ms p95), across 257 languages and across repos via canonical contract IDs — with a framework synthesizer that recovers DI/RPC/topic edges and a `resolution_outcomes` signal that tells you where it couldn't. The honest caveat stands: no static call graph resolves reflection, `eval`, or codegen, and Gortex says so rather than pretending otherwise. If your agent acts on what it traces, that honesty is the point.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
