---
title: "Gortex: What Changed and How It Compares to Alternatives"
description: "A follow-up on Gortex after a week of heavy development: 92 languages, daemon mode, GCX1 wire format, LSP edge tiers, cumulative cost tracking — plus an honest comparison with repomix, Cursor's built-in indexing, GitNexus, and Sourcegraph."
date: 2026-04-19
lastmod: 2026-04-19
draft: false
slug: "gortex-update-and-comparison"
keywords: ["gortex", "code intelligence mcp", "gitnexus alternative", "repomix alternative", "sourcegraph alternative", "cursor codebase indexing", "mcp server comparison", "ai agent code context", "gortex daemon", "92 languages mcp"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

A lot happened in a week. When I wrote [the first Gortex article](/gortex/from-gitnexus-to-gortex/), the tool was at 33 languages and 48 MCP tools. Now it's at 92 languages and 44 tools (some got consolidated), with a daemon mode, a compact wire format, LSP-enriched edge accuracy tiers, and a cost tracker that tells you how many dollars you've avoided in AI API spend.

This post covers what changed and — more usefully — how Gortex actually compares to the other tools you might reach for.

## What's new

### 92 languages

The previous count was 33. In a few days of parser work it jumped to 92. The additions cover template engines (Blade, EJS, Jinja, Twig, ERB, Liquid, Pug, Handlebars), blockchain languages (Solidity, Move, Cairo, Noir), scientific computing (Julia, R, MATLAB, Fortran, COBOL, Ada, Pascal), emerging languages (Mojo, Odin, V, Carbon, Gleam), shell variants (PowerShell, Zsh, Fish), and a long tail of everything else.

At 92 languages the question "does it support my stack?" is basically answered for any non-esoteric project.

### Daemon mode

Previously, each editor window started its own Gortex process. If you had Claude Code and Cursor open on the same repo, you had two separate indexes. Now there's a daemon:

```bash
gortex daemon start --detach
gortex daemon install-service   # LaunchAgent on macOS, systemd --user on Linux
```

All editors connect as thin stdio proxies. One graph, shared across every tool. The daemon stays running across editor restarts and does live fsnotify watching on every tracked repo. It also handles branch switches intelligently — instead of re-indexing everything after a `git checkout`, it runs `git diff --name-status` and patches only what changed.

### gortex install vs gortex init

Setup is now split in two. `gortex install` runs once per machine and writes user-level artifacts: MCP config, skills, slash commands, hooks. `gortex init` runs once per repo and writes repo-level stuff: `.mcp.json`, `CLAUDE.md`, per-community skill files, and routing blocks in every detected agent's instructions file.

This means the tool-usage guidance isn't duplicated into every repository anymore. One copy per user, not one per repo.

### GCX1 wire format

The MCP tools now support an opt-in compact wire format. Pass `format: "gcx"` on any of the 13 supported tools and the response comes back in a more compact representation. Measured across a 20-case benchmark: **median −27.4% tiktoken savings** compared to JSON, best case −38.3%. That's on top of the existing ~94% reduction from not reading whole files. A TypeScript decoder is on npm as `@gortex/wire`.

### LSP-enriched edge accuracy

Every call graph edge now carries an `origin` field with a confidence tier: `lsp_resolved`, `lsp_dispatch`, `ast_resolved`, `ast_inferred`, `text_matched`. Tree-sitter alone gets you 70–85% accuracy on call resolution. LSP providers (go/types, SCIP) push that to 95–100%.

Tools like `get_callers` and `find_implementations` now accept a `min_tier` parameter. If you're doing a high-stakes refactor where a missed caller means a runtime error, set `min_tier: "lsp_resolved"` and only get back compiler-verified edges.

### Cumulative cost tracking

```bash
gortex savings
```

This command shows how many tokens Gortex has saved across all sessions, mapped to actual dollar costs per model (Claude Opus, Sonnet, Haiku, GPT-4o, GPT-4o mini). It reads from `~/.cache/gortex/savings.json` which persists across restarts. Token counts now use tiktoken (`cl100k_base`) instead of the old `chars/4` approximation — same tokenizer Claude and GPT-4 actually use.

### 15 IDE integrations

Six more agents added since the last article: Codex CLI, Gemini CLI, Zed, Aider, Kilo Code, OpenClaw. `gortex install` and `gortex init` auto-detect all 15 and configure only the ones present on the machine.

### Supply chain security

Pre-built binaries for linux/amd64, linux/arm64, darwin/amd64, darwin/arm64. Every release is cosign-signed, has SLSA-3 provenance, and is VirusTotal-scanned. Homebrew tap: `brew install zzet/tap/gortex`. Debian, RPM, and Alpine packages available too.

### Scale

Benchmarked on Apple Silicon:

| Repo | Files | Index time | Peak heap |
|------|------:|----------:|----------:|
| torvalds/linux | 70,333 | ~3 min | 5.07 GB |
| microsoft/vscode | 10,762 | ~1 min | 580 MB |
| zzet/gortex | 430 | 3.4s | 52 MB |

The Linux kernel runs. Most repos are nowhere near that.

---

## How it compares

There are several ways to give an AI agent context about a codebase. Here's an honest look at each.

### repomix

repomix packs a repository into a single text file and puts it in the context window. Simple, works with any AI tool, zero setup.

It works well for small repos and one-off tasks. The agent sees everything at once, so it can answer questions without tool calls. For repos under a few thousand files, it's often the right answer.

The problems show up as the codebase grows. Packing 50,000 files into context is not practical — you'd exceed every context window and spend a lot on tokens. It also doesn't help with ongoing sessions: every new conversation starts with a full pack. There's no graph, no cross-repo queries, no contract detection, no watch mode.

Use repomix for small repos and quick tasks. Use something graph-based for large codebases or sustained agent sessions.

### Cursor's built-in indexing (`@codebase`)

Cursor indexes your repo in the background. You can ask questions with `@codebase` and it retrieves relevant code. Zero setup, automatic.

For typical day-to-day work this is good enough. The agent can navigate the codebase reasonably well without any additional tooling.

The limits: the agent gets retrieval results, not a queryable graph with an MCP tool surface. It can't do blast radius analysis, contract detection, or dead code analysis as structured calls. There's no cross-repo support. No guard rules. No way to expose custom graph queries to the agent. It's also opaque — you can't see what confidence level the edges have or know when the index is stale.

If you're working on a single-language single-repo project with moderate complexity, Cursor's built-in indexing might be all you need. If you're doing multi-repo work, polyglot stacks, or microservices where you want to verify contracts hold across services, you need something more structured.

### GitNexus

[GitNexus](https://github.com/abhigyanpatwari/GitNexus) is the tool Gortex started as a response to. Covered in [the first article](/gortex/from-gitnexus-to-gortex/).

Short version: TypeScript, 8 languages, 16 MCP tools, persistent LadybugDB storage, hosted web explorer. The hosted explorer at gitnexus.vercel.app is a genuine advantage — zero-friction graph sharing without local setup.

| | GitNexus | Gortex |
|---|---|---|
| Languages | 8 | 92 |
| MCP tools | 16 | 44 |
| Storage | Persistent (LadybugDB) | In-memory + disk snapshot |
| IDE integrations | — | 15 |
| Inline editing | — | Yes |
| Contract detection | — | Yes |
| LSP edge accuracy | — | Yes |
| Daemon mode | — | Yes |
| Hosted explorer | Yes | — |
| Self-hosted only | — | Yes |

GitNexus is easier to get started with if you don't want to run infrastructure. If you need language coverage, tool surface area, or multi-repo support, Gortex covers more.

### Sourcegraph / Cody

Sourcegraph is battle-tested at enterprise scale — large organisations with hundreds of repos, complex access control, and dedicated search infrastructure. Cody is their AI assistant layer on top.

The differences are philosophical. Sourcegraph is built for humans browsing code. The search, the web UI, the navigation — all optimised for a developer reading code in a browser. Agent token efficiency is not the primary goal.

Gortex is built specifically for AI agents: tool surface designed for MCP, responses optimised for token count, guard rules to prevent agent mistakes, blast radius to predict impact. There's no SaaS option for Gortex — you run it yourself, it stays local, no code leaves your machine.

If you're at a large organisation that needs hosted infrastructure, SSO, and cross-team code search for humans, Sourcegraph is the right answer. If you're a developer who wants an agent-optimised code graph that runs on your laptop, Gortex is the right answer. They're not competing for the same use case.

---

## Honest tradeoffs

**When Gortex is not the right choice:**

- You have a small project. For anything under a few thousand files, repomix or Cursor's built-in indexing is enough.
- You want zero setup. `gortex install` then `gortex init` is about ten minutes, but that's still ten minutes.
- You need a hosted option. Gortex is self-hosted only right now — but hosted support is on the priority roadmap. Check [github.com/zzet/gortex](https://github.com/zzet/gortex) for the current status when you read this.
- You're in a large enterprise with compliance requirements. Gortex takes supply-chain security seriously — every release is cosign-signed with SLSA-3 provenance, and you can verify the binary hasn't been tampered with. But access control, audit logs, and commercial support contracts don't exist yet. If that's a blocker and you're evaluating Gortex for your organisation, reach out directly at [license@zzet.org](mailto:license@zzet.org).

**When Gortex earns its setup time:**

- **The token savings are real.** Here's `gortex savings` from my own usage after one week — and not an optimized week, early sessions were still dialing in the setup:

  ```
  Calls counted:   599
  Tokens returned: 219,720
  Tokens saved:    2,979,097
  Efficiency:      14.6x

  Cost avoided (tokens saved × input-price, USD):
    claude-haiku-4.5   $2.98
    claude-opus-4      $44.69
    claude-sonnet-4    $8.94
    gpt-4o             $7.45
    gpt-4o-mini        $0.45
  ```

  599 tool calls. 2.9M tokens saved. At Opus pricing, that's $44 not spent in a single week. With things properly dialed in, my current usage runs at least $50–60 saved per week — which, compared to a $200/month Claude subscription, is a serious number. And that's before accounting for context window preservation: fewer tokens consumed per session means longer, more coherent sessions before hitting the limit. For anyone on a plan with usage caps, that matters in a way money can't fix — you can upgrade your subscription, but you can't buy back the hour you spent waiting for the limit to reset.

- **Polyglot stack.** 92 languages means the whole project is visible — Go services, Ruby scripts, HCL infrastructure, Protobuf schemas, all in one graph. Most tools cover eight languages and ignore the rest.

- **Multiple repos.** You work on a backend, a frontend, and a shared library. With cross-repo mode, blast radius analysis spans all three. Contract detection finds where your Go API's routes don't match what the TypeScript client expects, across repo boundaries.

- **Large codebases.** The Linux kernel indexes in ~3 minutes at 70K files. VSCode in ~1 minute. Incremental restarts after that are ~200ms.

- **Inline editing without reading files.** `edit_symbol` and `rename_symbol` let the agent patch code directly by symbol ID. No read → find → rewrite-whole-file loop. This cuts the per-edit tool call count significantly on multi-file refactors.

- **Guard rules.** You define dependency boundaries in `.gortex.yaml`. The agent can't accidentally import infrastructure into a domain package without getting a flag before review. Useful on any codebase where architecture discipline matters.

- **Agent config you can trust.** `audit_agent_config` validates your `CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, and similar files against the live graph — finds stale symbol references and dead file paths that accumulate as code evolves. Nothing else does this.

- **Multiple team members on different editors.** One daemon, one graph, 15 agents configured from two commands. Everyone on the team — whether they use Claude Code, Cursor, Kiro, or Copilot — works against the same index.

- **High-stakes refactors.** LSP-enriched edges carry confidence tiers. Filter `get_callers` to `min_tier: "lsp_resolved"` when you need compiler-verified results, not heuristic matches. 95–100% accuracy on call resolution instead of 70–85%.

---

Source: [github.com/zzet/gortex](https://github.com/zzet/gortex)

```bash
brew install zzet/tap/gortex   # macOS
gortex install                  # once per machine
gortex init                     # once per repo (optional)
```
