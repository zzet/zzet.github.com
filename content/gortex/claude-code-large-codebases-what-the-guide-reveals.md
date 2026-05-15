---
title: "Anthropic's large codebase guide is a Gortex tutorial in disguise"
description: "Anthropic published a best-practices guide for Claude Code in large codebases. It's a good guide. It also describes, step by step, exactly what Gortex ships out of the box — and reveals the real cost of the grep-based approach it defends."
date: 2026-05-15
lastmod: 2026-05-15
draft: false
slug: "claude-code-large-codebases-what-the-guide-reveals"
keywords: ["gortex", "claude code large codebase", "claude code best practices", "claude code mcp server", "code intelligence mcp", "claude code harness", "claude code lsp", "gortex vs claude code", "ai coding agent context", "claude code skills", "claude code hooks", "gortex daemon", "mcp tools codebase", "multi-repo code intelligence", "service boundary bugs", "cross-repo mcp"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

After reading [How Claude Code works in large codebases: Best practices and where to start](https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start), Anthropic's new "Claude Code at scale" series opener, I felt a half-told story. The guide is worth reading — the observation that the harness matters as much as the model is correct, the LSP recommendation is the right call, and the skills/progressive-disclosure framing is exactly right.

<lead>It's also, inadvertently, a detailed description of what Gortex ships out of the box.</lead>

This isn't a gotcha. The guide describes a real problem and real solutions. The challenge is that it presents those solutions as things engineering teams must build, staff, and maintain — a dedicated DRI, quarterly configuration reviews, per-language LSP setups, custom MCP servers. What it doesn't mention is that a large fraction of that infrastructure already exists. And after spending years in multi-team environments where preferences and standards drift constantly, I've learned that documentation describing what the harness should look like is much less useful than a harness that already works.

---

## The grep tradeoff the guide doesn't fully name

The guide opens with a reasonable defense of grep-based agentic search over stale RAG embeddings:

> "Claude Code navigates a codebase the way a software engineer would: it traverses the file system, reads files, uses grep to find exactly what it needs, and follows references across the codebase."

Stale embedding indexes are a real failure mode — agreed. But grep across a large codebase isn't free either:

- Every grepped file is a file read.
- Every file read consumes context.

When Claude greps for a function name and gets three thousand matches, it burns context narrowing down which ones matter — before it's touched the actual problem. The guide acknowledges this obliquely:

> "If you ask it to find all instances of a vague pattern across a billion-line codebase, you'll hit context-window limits before the work begins."

The proposed fix is discipline: layered CLAUDE.md files, LSP integration, `.ignore` rules, codebase maps. All sound. All also compensating for the fundamental choice of navigating by traversal rather than by graph.

What the guide doesn't name is a third option: a persistent, live graph that patches itself on file save, serves structured symbol queries instead of text matches, and never needs to be re-indexed from scratch. That's the gap the article works around without naming it.

---

## "The harness matters as much as the model"

This is the guide's strongest observation. It lists seven components every serious deployment needs:

**CLAUDE.md files** — Root file for the big picture, subdirectory files for local conventions, kept lean. Don't put reusable expertise there (that belongs in a skill).

**Hooks** — Scripts that run at key lifecycle moments. A start hook loads team-specific context dynamically; a stop hook reflects on the session and proposes CLAUDE.md updates while the context is fresh.

**Skills** — Packaged instructions for specific task types, loaded on demand. The security review skill loads when relevant; the deployment skill activates in the payments directory.

**Plugins** — Bundled skills, hooks, and MCP configs distributed across the org so new engineers get the same setup as veterans on day one.

**LSP integrations** — Symbol-level navigation so Claude follows a reference to its definition rather than grepping for a name. One enterprise company deployed LSP org-wide before their Claude Code rollout, specifically for C and C++ at scale. Good call.

**MCP servers** — Structured search tools Claude can call directly. The guide says "the most sophisticated teams" build these. This is probably the most interesting part, tho.

**Subagents** — Isolated Claude instances for exploration work, so mapping and editing don't compete for the same context window.

Yes, this is good architecture. It's also a description of Gortex.

`gortex init` writes the CLAUDE.md hierarchy, layered by directory, with initial content generated from the actual graph — not filled in by hand. `gortex init --skills` generates per-community SKILL.md files clustered by Louvain algorithm, one per functional area of your codebase. The PreToolUse, PreCompact, and Stop hooks ship pre-wired. The 64 MCP tools are the structured search layer the guide calls "the most sophisticated teams" build. The graph resolves symbols in 256 languages without requiring a language server binary for each one.

The guide describes what you need to build. Gortex is the shortcut to having it built.

---

## On LSP — with, not instead

The guide's LSP recommendation is correct and worth emphasizing:

> "Grep for a common function name in a large codebase returns thousands of matches and Claude burns context opening files to figure out which matters. LSP returns only the references that point to the same symbol, so the filtering happens before Claude reads anything."

This is exactly right. Symbol-level navigation matters, and LSP is a proven way to get it. Gortex works with LSP too — the code intelligence plugin wires them together so Claude has both the graph's breadth and LSP's compiler-verified precision for the languages that have a mature server.

But LSP has a structural boundary: it's single-language and single-repo by design. A Go language server knows every call site of a function within a Go codebase. It does not know that a TypeScript client in a different repo is fetching the HTTP endpoint that function serves. It doesn't see across the Kafka topic boundary between a Java producer and a Python consumer. It doesn't know that the protobuf schema updated in repo A is imported by three services in repos B, C, and D.

This is where Gortex picks up where LSP stops. The graph layer cross-references imports, symbols, HTTP routes, message topics, and gRPC definitions across every tracked repo simultaneously. LSP is great at "what calls this function?" Gortex answers "what, across the entire system, depends on this contract?"

---

## The bugs that live at boundaries

The guide evaluates Claude Code as if a codebase is one repository. That's been increasingly not true for a while.

Most production systems today are a Go API, a TypeScript frontend, a Python data pipeline, maybe some Rust for the hot path, sharing protobuf definitions over a contract that nobody owns explicitly. The bugs that cause incidents don't usually live inside a single service. They live at the boundaries: the renamed route nobody updated in the client, the Kafka message schema that drifted between producer and consumer, the shared library function that changed its error semantics in a way the callers didn't expect.

A language server can't see across these boundaries. Neither can grep. Neither, frankly, can a carefully written CLAUDE.md file, because the boundary is dynamic — it changes every time someone updates the producer without remembering to tell the consumers.

Gortex's contract layer exists specifically for this. `get_contracts` extracts HTTP routes from gin, Express, FastAPI, Spring, and others; gRPC service definitions; GraphQL schemas; Kafka and NATS topics; WebSocket events; env var dependencies. It normalizes them to canonical IDs — `http::GET::/api/users/{id}` is the same contract whether it's defined in Go and consumed in TypeScript. `check_contracts` then finds providers with no consumers and consumers with no matching provider, across every tracked repo in one call.

The kind of mismatch this catches — route renamed in the server, client still calling the old path — usually only surfaces at deployment time. Catching it in the graph, before the code leaves the editor, is a different class of safety property than anything grep or LSP provides.

---

## The freshness problem, applied to skills

The guide raises a real concern about CLAUDE.md files:

> "As models evolve, instructions written for your current model can work against a future one. CLAUDE.md files that guided Claude through patterns it used to struggle with may either become unnecessary or actively constraining when the next model ships."

This is true of hand-written files generally — they drift from the codebase they describe. The guide recommends quarterly reviews to keep them accurate.

Gortex's community skill files are generated from the live graph. When a new service gets extracted from a monolith, when a dependency gets updated, when the package structure changes after a big refactor — `gortex init --skills` regenerates from the current graph state. The skills describe the actual code, not the code as someone remembered it when they wrote the file last quarter.

This doesn't eliminate CLAUDE.md. Project-specific conventions, external integration caveats, organizational context — those still belong in a hand-written file. But the structural layer, the "what lives where and how the pieces connect" part, shouldn't need a calendar reminder to stay accurate. It should update when the code does.

---

## What the day actually looks like

The guide is honest about the organizational cost:

> "An emerging role in several organizations is an agent manager: a hybrid PM/engineer function dedicated to managing the Claude Code ecosystem."

And:

> "Teams should expect to do a meaningful configuration review every three to six months."

This is an accurate picture of Claude Code at scale without a pre-built code intelligence layer. Someone has to own the harness, someone has to keep it current, someone has to distribute it when a new engineer joins.

Here's the same timeline with Gortex.

**Day 0**

```bash
curl -fsSL https://get.gortex.dev | sh
gortex install
gortex daemon start --detach
gortex init
```

Every editor on the machine gets the MCP config. The CLAUDE.md is written. The PreToolUse hook intercepts grep and routes it through the graph. The daemon holds the live graph and patches it when files change.

**Day 1 and forward**

Branch switches reconcile via `git diff`, not a full re-index. `gortex enrich blame` stamps last-author on every symbol; `gortex enrich coverage` connects the coverage profile. `gortex track /path/to/new-service` adds a new service to the graph in the background — the agent queries it immediately.

**Quarterly reviews**

The graph doesn't drift. The structural knowledge updates when the code does. What's left for the quarterly review is the stuff that should be hand-written: conventions, external dependencies, organizational context. That's a much shorter document than a full codebase map.

The difference isn't features. It's carrying cost. The guide describes a flywheel that teams have to build. Gortex shifts where that work happens — from your DRI's calendar to the graph engine, running in the background, always warm.

---

**Without a code intelligence layer.** You ask Claude to trace a bug across two services. It greps, opens files, greps again, opens more files, runs out of context before finding the cause. You restart, give it a more specific starting point, try again. The cost isn't the token spend — it's the interruption. You're debugging the agent's navigation instead of the bug.

**With the harness the guide describes.** You have CLAUDE.md files that give Claude a map, LSP so it can follow references, skills so the right context loads. Genuinely better. The agent completes more tasks in fewer turns. The cost: someone built and maintains that infrastructure.

**With Gortex.** The agent calls `smart_context` instead of opening files — definition, callers, callees, type relationships, community cluster, in one call, at a fraction of the token cost. It calls `flow_between` to trace data through the graph without reading intermediate files. It calls `verify_change` to confirm a refactor doesn't break callers. It calls `check_contracts` to validate that the renamed route doesn't orphan the TypeScript client two repos over. It can edit a symbol without reading the file first.

The session doesn't end because Claude ran out of context building a map. It ends because the work is done.

The time saved isn't just the token cost. Waiting for a context reset, rebuilding the agent's understanding of where it was — that's dead time. You can buy more tokens; you can't buy back the hour.

---

## What the guide gets right

The harness-first thesis is correct. A capable model with no context about where it's working is still a capable model guessing. The extension layer — skills, hooks, plugins, MCP — is the right abstraction.

The progressive disclosure framing for skills is exactly right. Not everything should be in every session. Load what's relevant to the task, offload what isn't.

The governance section is honest. Large organizations do need cross-functional working groups early. Who controls the plugin marketplace, how AI-generated code goes through review — these questions don't have easy answers and it's better to surface them before the rollout than after.

The recommendation to initialize in subdirectories rather than the repo root is a useful heuristic that most teams get wrong the first time.

---

## Bottom line

Anthropic's guide describes the right destination: a live, symbol-aware harness that gives the agent structured context instead of a filesystem to crawl. Where it differs from Gortex is in who builds it and what it covers.

The guide assumes: your team builds the harness, per-language LSP servers cover symbol navigation, CLAUDE.md files map the codebase, quarterly reviews keep it current, and a dedicated DRI holds it together.

Gortex assumes: the harness ships pre-built, the graph covers 256 languages without per-language server setup, the skills regenerate from live graph state, the daemon keeps context warm, and the cross-repo contract layer handles the boundary bugs that LSP and grep structurally can't see.

The gap isn't a criticism of the guide. It's a description of what's left to do after you follow the guide — and how much of that is already solved.

Source: [github.com/zzet/gortex](https://github.com/zzet/gortex)

```bash
curl -fsSL https://get.gortex.dev | sh
gortex install
gortex daemon start --detach
gortex init
```
