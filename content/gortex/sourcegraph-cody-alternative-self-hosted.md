---
title: "Sourcegraph alternative self-hosted: a Cody replacement for AI agents"
description: "A Sourcegraph alternative self-hosted for AI agents: how OpenGrok, Zoekt, Greptile, and an Apache-2.0 code graph compare on cost, ops, and MCP."
date: 2026-05-24
lastmod: 2026-06-06
draft: false
slug: "sourcegraph-cody-alternative-self-hosted"
keywords: ["Sourcegraph alternative self-hosted", "Cody individual tier deprecated alternative", "Greptile alternative open source", "OpenGrok vs Zoekt", "self-hosted code search engine", "open source sourcegraph alternative"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You wired up Cody, your team got used to whole-codebase context in chat, and then the plan disappeared. Cody Free and Cody Pro were [discontinued on July 23, 2025](https://sourcegraph.com/blog/changes-to-cody-free-pro-and-enterprise-starter-plans), with new signups ending June 25, 2025. The Enterprise Starter plan no longer bundles Cody. Individuals were pointed at Amp, a separate agentic product. If you wanted self-hosted code intelligence without an enterprise contract, the door closed.

That is the search behind this article: a **Sourcegraph alternative self-hosted** that an AI agent can actually call, runs on your own machine, and doesn't require a custom quote or a Kubernetes cluster. The honest news is that there are several good options depending on what you need — and they are not interchangeable. This piece maps them fairly first, then shows where a local, MCP-native code graph fits.

> **Short answer:** For raw code search at monorepo scale, [Zoekt](https://github.com/sourcegraph/zoekt) (Go, Apache-2.0) is the fastest open-source engine, and [OpenGrok](https://github.com/oracle/opengrok) gives you a mature browse-and-cross-reference web UI. Both are search/browse tools with no AI-agent surface. [Greptile](https://www.greptile.com/pricing) is proprietary and cloud-first, scoped to PR review. If you want self-hosted, free, *agent-native* code intelligence — usages, callers, call chains, impact analysis, context packing, all over MCP — Gortex is the closest fit: Apache 2.0, in-process (no DB, JVM, or cluster), 100+ MCP tools, 257 languages. Sourcegraph itself is now enterprise-only with a private core repo, so it is not realistically an individual or small-team option.

## Why Sourcegraph and Cody stopped being an individual option

Sourcegraph is genuinely the category leader at org scale. It does precise SCIP/LSIF cross-repo go-to-definition and find-references across thousands of repos, Batch Changes for mass automated refactors, Code Insights dashboards, and RBAC/SSO/audit. If you run hundreds of engineers across a large monorepo estate, none of the alternatives below match that depth. That concession is the whole point — overclaiming parity at that scale would be dishonest.

The friction is access and operability, not capability. Two changes pushed individuals and small teams out:

- **The free individual tier went away.** Cody Free and Cody Pro were discontinued on 2025-07-23. Sourcegraph itself is enterprise-only with custom quotes; there is no self-serve individual plan.
- **The core is no longer source-available.** Sourcegraph relicensed Code Search from Apache to its non-open-source "Sourcegraph Enterprise" license [on June 13, 2023](https://en.wikipedia.org/wiki/Sourcegraph), then [made the core repository private on August 22, 2024](https://devclass.com/2024/08/21/sourcegraph-makes-core-repository-private-co-founder-complains-open-source-means-extra-work-and-risk/). Co-founder Quinn Slack argued open source adds "extra work and risk" for full server-side end-user applications. So you cannot call Sourcegraph open source today — and self-hosting it means operating a multi-service cluster.

Amp, the suggested migration path, is a different shape of product. Its free tier is [ad-supported and requires opting into training-mode data sharing](https://tessl.io/blog/amp-s-new-business-model-ad-supported-ai-coding/) — your thread data is used for training to use the free tier. "Free" is doing a lot of work there. If your reason for self-hosting was control over where your code goes, that is the wrong direction.

## The self-hosted code-search menu, fairly

Strip away the AI layer and you land in classic open-source code search. These tools are mature and worth knowing even if you never touch Gortex.

### OpenGrok: mature browse UI, but JVM-heavy and no AI surface

[OpenGrok](https://github.com/oracle/opengrok) is a source-code search and cross-reference engine written in Java, with a polished web UI, history, and diffs, deployed at large orgs for years. For a human browsing an unfamiliar codebase, it is excellent and battle-tested (roughly 4.8k GitHub stars as of mid-2026).

Two caveats matter for this job. First, operability: per the [official docs](https://oracle.github.io/opengrok/) (as of 2026), OpenGrok needs a Java runtime (17–21), a servlet container such as Apache Tomcat 10.x+ or GlassFish, and Universal Ctags, and the indexer is typically run with about an 8 GB JVM heap. That is a real stack to stand up and keep alive. Second, it is [CDDL-1.0 licensed](https://en.wikipedia.org/wiki/OpenGrok) (a weak-copyleft, MPL-derived license, not Apache), and — critically — it has zero AI/LLM/MCP surface. It is a human web tool, not an agent tool.

### Zoekt: fastest raw search, but search-only with no native MCP

[Zoekt](https://github.com/sourcegraph/zoekt) is the fastest open-source raw code search: Go, Apache-2.0, trigram-based substring and regex, [sub-50ms on Android-scale corpora](https://deepwiki.com/sourcegraph/zoekt) (~2 GB of text), symbol-aware ranking, a web UI plus JSON/gRPC APIs. It is the engine Sourcegraph uses internally (~1.7k stars as of mid-2026).

Its limit is by design: Zoekt is search-only. There is no graph, no cross-reference navigation, no callers/usages, no impact analysis. And there is no first-party MCP — only an unofficial community bridge, [itaymendel/zoekt-mcp](https://github.com/itaymendel/zoekt-mcp). So an agent can grep your code fast, but it cannot ask "who calls this?" or "what breaks if I change this signature?"

### Greptile: AI PR review, but proprietary, cloud-first, per-review priced

[Greptile](https://www.greptile.com/pricing) is an AI code-review product — review comments on pull requests, GitHub/GitLab only. It is a different category: a review bot, not a general code-intelligence engine for your agent. Self-hosting is Enterprise-only (your own AWS, bring-your-own-LLM); in cloud mode your code passes through Greptile's servers. Open-source projects get free usage; everyone else does not.

Pricing is the other catch. As of 2026 the Cloud plan is [$30 per seat per month including 50 reviews, with extra reviews at $1 each](https://costbench.com/software/ai-code-review/greptile/) — a [per-review model that drew criticism](https://www.agent-wars.com/news/2026-05-01-greptile-per-review-pricing) for penalizing high-throughput, agent-generated PR volume. (Vendor pricing changes often; check current terms.) Greptile reports v4 (early 2026) improved detection accuracy, but those are vendor-internal numbers. If your goal is feeding an agent deep repository context, a PR-review SaaS is the wrong tool regardless of price.

### The honest scorecard

Each incumbent misses at least one of the three things this search is really about: open license, low-ops self-hosting, and a native AI-agent (MCP) surface.

| | Sourcegraph | OpenGrok | Zoekt | Greptile | Gortex |
|---|---|---|---|---|---|
| **Hosting** | Self-host cluster or single-tenant cloud | Self-host | Self-host | Cloud-first (self-host = Enterprise) | Self-host, local-first |
| **Cost** | Enterprise custom quote | Free | Free | $30/seat + $1/review | Free |
| **License** | Proprietary (core private) | CDDL-1.0 | Apache-2.0 | Proprietary | Apache-2.0 |
| **AI-agent / MCP** | Cody (enterprise-bundled) | None | Unofficial community bridge | Cloud review bot | Native, 100+ MCP tools |
| **Beyond search** | Full graph, Batch Changes, Insights | Cross-ref browse | Search only | PR review | Graph: usages, callers, impact |
| **Languages** | Many (SCIP/LSIF) | Many (Ctags) | Many | Many | 257 |
| **Cross-repo** | Precise, org-scale | Limited | Per-index | Per-PR repo | Yes, API-contract checks |
| **Operability** | Multi-service cluster | JVM + Tomcat + ~8 GB heap | Go binary(s) | SaaS dependency | One signed binary + daemon |

Read the table honestly: Sourcegraph wins beyond-search depth and org-scale navigation; Zoekt wins raw search speed and is genuinely Apache-licensed; OpenGrok wins human browsing maturity. The open gap — Apache license *and* low-ops self-host *and* a native agent surface *and* a graph rather than just search — is where Gortex sits.

## Gortex: the Sourcegraph alternative self-hosted for AI agents

[Gortex](https://github.com/zzet/gortex) is an Apache-2.0 code graph and intelligence engine. It indexes your repositories into an in-memory knowledge graph and exposes that graph over a CLI, an MCP server, and a web UI. It is built to hand an AI coding agent the right code with the fewest tokens, deterministically. If you want the conceptual background, start with [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/).

Three properties line up with exactly what fell out of the table above.

**Apache 2.0, local-first, no data-sharing strings.** It runs entirely on your machine. There is no API key required to start and no cloud round-trip — your code stays put. That is a cleaner "free" than an ad-supported tier that wants your threads for training, or a per-review meter. The full case for keeping this on your own hardware is in [local-first, open-source code intelligence — no cloud, no API key](/gortex/local-first-code-intelligence-no-cloud/).

**In-process, zero external dependencies.** No database, no JVM, no servlet container, no Kubernetes. It is one signed binary plus a daemon that indexes your repos and serves your agent. Compare that to standing up OpenGrok's JVM + Tomcat + ~8 GB heap, or operating a Sourcegraph cluster. The operability difference is the practical reason small teams can actually run this.

**MCP-native, with a graph instead of just grep.** This is the part Zoekt and OpenGrok structurally cannot do. Gortex exposes 100+ MCP tools, and one install auto-configures every detected agent across 15 integrations (Claude Code, Cursor, Windsurf, VS Code/Copilot, Continue, Cline, and more). The agent gets real graph operations:

- `find_usages` — every reference, each tagged with a typed context (call, return_type, field, value), zero false positives from comments or strings.
- `get_callers` / `get_call_chain` — reverse and forward call graphs across files.
- `explain_change_impact` / `get_dependents` — risk-tiered blast radius. The precomputed depth-3 reach index makes this sub-millisecond (measured impact p95 = 0.01ms), so an agent can ask "what breaks?" on every edit.
- `smart_context` — a task-aware minimal working set in one call, replacing 5–10 exploration round-trips.
- `contracts` — cross-repo API-contract checks (HTTP routes, gRPC, GraphQL, message topics, env vars) matched provider↔consumer for orphans and mismatches.

A first install looks like this:

```bash
curl -fsSL https://get.gortex.dev | sh
gortex track ./your-repo
gortex daemon start --detach
# detected agents are now wired to the MCP server; no cluster, no DB
```

On retrieval cost, the graph pays off in tokens. Reading a symbol via `get_symbol_source` returns ~80% fewer tokens than reading the whole file, and across BENCHMARK.md's query set Gortex lands **3–50× fewer tokens per response** versus ripgrep-plus-full-read (peak ~50× on identifier lookups), at median recall@2k of 1.00 versus 0.00 for ripgrep. That is the difference between grep (fast text, no meaning) and a resolved graph (typed edges an agent can trust). The deeper story on why agents need this is in [the code-intelligence MCP server your setup is missing](/gortex/mcp-server-for-codebase-context/).

### Where Gortex is the wrong tool

The house style here is to tell you where this loses, so the rest is credible.

- **Org-scale precise navigation is Sourcegraph's, not Gortex's.** Batch Changes, Code Insights, SCIP/LSIF precision across thousands of repos, and audited human code review with RBAC are mature there and not the target here. Gortex targets the individual/small-team + AI-agent, local-first, no-contract job.
- **It is a binary + daemon, not a browser tool.** Heavier to bootstrap than a WASM/in-browser option; there is no in-browser story.
- **Some refactors are Go-only for now.** `move_symbol` and `inline_symbol` are Go-only at present; `rename_symbol` (cross-file) and `safe_delete_symbol` are broader.
- **Default embeddings are lightweight.** The default semantic layer is a baked GloVe-50d model (zero setup, no API key); for stronger semantic recall you opt into MiniLM/Ollama/OpenAI. Gortex's edge is hybrid retrieval plus a *deterministic* graph (BM25 + vector + RRF + graph-centrality), not semantic supremacy — exact-symbol recall is where it dominates (bm25 R@5 = 96.8%), concept/multi-hop recall is modest.

If you are weighing it against other small, local options rather than the enterprise incumbents, the head-to-head is in [local code-graph MCP servers compared: codanna, ChunkHound, Serena, Context7](/gortex/local-code-graph-mcp-servers-compared/).

## FAQ

### What is the best self-hosted Sourcegraph alternative in 2026?

It depends on the job. For raw code search at monorepo scale, Zoekt (Go, Apache-2.0) is the fastest open-source engine, and OpenGrok offers a mature browse-and-cross-reference web UI — but both are search/browse tools with no AI-agent surface. If you want self-hosted, free, agent-native code intelligence — usages, callers, call chains, impact analysis, context packing over MCP — Gortex is the closest fit: Apache 2.0, in-process, 100+ MCP tools, 257 languages. Sourcegraph itself is now enterprise-only with a private core repo, so it is not an option for individuals or small teams.

### Cody's individual (Free/Pro) tier was deprecated — what should I use instead?

Cody Free and Cody Pro were discontinued on July 23, 2025, and Sourcegraph pushed individuals to Amp, whose free tier is ad-supported and requires opting into training-mode data sharing. If you want a free, local alternative without those strings, Gortex is a self-hosted, Apache-2.0 code-intelligence engine that plugs into your existing AI coding agent (Claude Code, Cursor, Windsurf, Copilot, Continue, and more) over MCP and runs entirely on your machine.

### OpenGrok vs Zoekt — which should I choose?

OpenGrok is the more full-featured human tool: a Java web UI with cross-references, history, and diffs — but it needs a JVM (17–21), a servlet container like Tomcat, Universal Ctags, and roughly an 8 GB indexer heap, and it is CDDL-licensed. Zoekt is the fastest raw search (Go, Apache-2.0, trigram regex, sub-50ms on Android-scale code) but is search-only with no cross-reference navigation. Neither has a native AI-agent (MCP) surface, which is where a graph tool like Gortex differs from both.

### Is there an open-source Greptile alternative?

Greptile is a proprietary, cloud-first AI code-review product (self-hosting is Enterprise-only, and as of 2026 it uses per-review pricing of $30/seat plus $1/review beyond 50), scoped to PR review on GitHub/GitLab. If you want an open-source, self-hostable code-intelligence engine instead — one that gives your agent deep repository context, usages, impact analysis, and cross-repo API-contract checks rather than just PR comments — Gortex is Apache 2.0, runs locally, and exposes everything over MCP.

### Do I need a database, JVM, or Kubernetes cluster to self-host code intelligence for AI agents?

No. Classic stacks add operational weight — OpenGrok needs a JVM plus a servlet container, and Sourcegraph runs as a multi-service cluster. Gortex is in-process with an in-memory graph and zero external dependencies, so there is no database, no JVM, and no cluster to operate — just one signed binary and a daemon that indexes your repos and serves your agent over MCP.

## Bottom line

The incumbents each solve part of this. Sourcegraph is the org-scale leader for precise cross-repo navigation, Batch Changes, and audited review — but it is enterprise-only and no longer source-available. OpenGrok and Zoekt are solid open-source search, but they are search/browse tools with no agent surface and (for OpenGrok) a heavy JVM stack. Greptile is a capable cloud review bot, not a general agent-context engine. The unfilled spot — Apache-2.0, local-first, low-ops, and MCP-native with a real graph — is the one Gortex is built for, aimed squarely at the individual and small-team AI-agent workflow that lost its free tier.

If that is your situation, it costs one command to try: [github.com/zzet/gortex](https://github.com/zzet/gortex).
