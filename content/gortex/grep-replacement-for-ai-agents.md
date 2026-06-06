---
title: "Code search for AI agents: the grep replacement is three tools, not one"
description: "Code search for AI agents needs three modalities, not one: lexical, structural, and graph. A field guide to when grep, ast-grep, and a code graph each win."
date: 2026-06-06
lastmod: 2026-06-06
draft: false
slug: "grep-replacement-for-ai-agents"
keywords: ["code search for AI agents", "semantic code search vs grep", "ast-grep vs grep", "structural code search tree-sitter query", "trigram code search vs grep", "grep replacement for ai agents"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

Watch an AI coding agent work and you will see the same move over and over: it shells out to `grep` (or `ripgrep`). Find a symbol? grep. Find the callers? grep. Find where a config value is read? grep. The agent reaches for grep because it is the only search tool that works on any repository, in any language, with no index, no API key, and no setup. That universality is exactly why **code search for AI agents** is harder than it looks — grep is the universal primitive, and most code questions are not text questions.

The result is the failure mode you have watched burn tokens: you ask "who calls `parseConfig`?", the agent greps `parseConfig(`, gets 40 hits — comments, a log-line string, a method on an unrelated type, the definition itself — and reads all 40 files to find the three real ones. The model did its job. The tool was wrong for the question.

> **Short answer:** There is no single grep replacement for AI agents — there are three search modalities, and a good agent needs all of them. *Lexical* search (grep/ripgrep/trigram/BM25) answers "where does this **text** appear." *Structural* search (ast-grep, tree-sitter queries) answers "where does this **code shape** appear." *Graph* search (resolved call/usage/implements edges) answers "who actually **calls or uses** this." Grep only does the first one, approximately.

This is a field guide to those three modalities — what each is good at and bad at, the 2026 evidence — before it gets to how [Gortex](https://github.com/zzet/gortex) hands an agent all three behind one interface. Stop reading after the modality section and you should still be able to pick the right search tool for any code question.

## Why AI agents default to grep (and why that is the wrong default)

Grep wins on the only axis that matters at agent-bootstrap time: zero prerequisites. Any directory is a valid corpus. Any language is fair game — grep does not parse, so it does not care whether your repo is Rust, COBOL, or a `.env` file. No index, no embedding model, no vector database. For an agent dropped into an unfamiliar repo a thousand times a day, that is decisive. `ripgrep` makes it fast too, via Rust plus a parallel thread pool, SIMD, gitignore-aware filtering, and *literal extraction* — pulling a guaranteed literal like `_SUSPEND` out of `[A-Z]+_SUSPEND` so it skips most bytes before the regex engine runs. It [searches the Linux kernel for `[A-Z]+_SUSPEND` in 0.082s versus git grep's 0.273s](https://burntsushi.net/ripgrep/).

The trap is confusing "works everywhere" with "right for everything." Grep matches characters in a byte stream. It has no idea what a function is, what a call is, or what scope a name lives in. So when the question is structural ("where do we build SQL by string concatenation") or relational ("who actually calls this, excluding comments and unrelated symbols"), grep answers a *different, easier* question — text co-occurrence — and the agent pays for the gap by reading and filtering noise. Cursor's team notes that [agents love grep and that `rg` invocations routinely take more than 15 seconds in large monorepos](https://cursor.com/blog/fast-regex-search), which is why they built a client-side n-gram index instead of leaning on raw `rg`.

## The three modalities of code search for AI agents

Think about code search by the *question* each modality can actually answer.

| Modality | The question it answers | Representative tools | Matches on |
|---|---|---|---|
| **Lexical** | Where does this **text** appear? | grep, ripgrep, Zoekt, BM25 | Characters / tokens |
| **Structural** | Where does this **code shape** appear? | ast-grep, tree-sitter queries | Parsed AST nodes |
| **Graph** | Who **calls / uses / implements** this? | LSP, code property graphs | Resolved edges between symbols |

Each modality knows more about code than the one above it and works on fewer things: lexical runs on anything but knows nothing about code; structural understands syntax but not meaning across files; graph understands resolved relationships but needs a real parse-and-resolve pipeline. Pick by the question, not by habit.

### Lexical: where does this text appear?

Grep's home turf, with three rungs of sophistication.

**Plain regex (grep/ripgrep)** scans files linearly on every query. No persistent state. Best for raw text, logs, and any unindexed corpus. A good floor.

**Trigram index (Zoekt, Sourcegraph)** pre-builds an index of every 3-character sequence plus byte positions, so a regex becomes index lookups instead of a linear scan. [Zoekt achieves sub-50ms search on ~2GB corpora like Android, ~10ms for rare strings when caches are warm](https://github.com/sourcegraph/zoekt); it is Apache-2.0 (as of 2026), with a CLI and indexserver for multi-repo scale. The cost is maintaining an index. ([Internals walkthrough.](https://thomastay.dev/blog/how-zoekt-works/))

**Ranked lexical (BM25)** stops treating search as a yes/no filter and starts *ranking* matches by term frequency, tuned for code. An agent does not want all 40 matches, it wants the best 5. Notably, Sourcegraph Cody [removed embeddings for an adapted BM25F over its code graph](https://sourcegraph.com/docs/cody/core-concepts/embeddings), citing third-party code transmission, vector-DB maintenance, and poor scaling past 100K+ repos — [keeping retrieval "boring and relevant"](https://sourcegraph.com/blog/keeping-it-boring-and-relevant-with-bm25f).

### Structural: where does this code shape appear?

Structural search matches *code structure*, not text. Tools like ast-grep parse a file into a tree-sitter AST and match patterns written like ordinary code, with `$METAVAR` wildcards for sub-expressions. The [tree-sitter query language is an S-expression (Lisp-like) DSL](https://tree-sitter.github.io/tree-sitter/using-parsers/queries/) for matching syntactic structures; ast-grep wraps that in a friendlier, code-shaped syntax.

Why this beats text-regex for code: it ignores whitespace and formatting, respects nesting, and — critically — never matches inside comments or string literals, because those are different AST nodes. A regex for `foo(` matches a call, a comment, and a docstring identically. An AST pattern for a call expression matches only the call.

```bash
# Text regex: matches calls, comments, and strings alike
rg 'exec\.Command\('

# Structural: matches only the call-expression shape, $CMD is a wildcard
ast-grep --pattern 'exec.Command($CMD)'
```

[ast-grep](https://github.com/ast-grep/ast-grep) ([how it works](https://ast-grep.github.io/advanced/how-ast-grep-works.html)) is MIT-licensed (as of 2026), written in Rust, runs multi-core, and is used by CodeRabbit, Vercel Turbo, and Vue tooling. It is the strongest tool for **codemods and large-scale rewrites** — when you need to *match a shape and transform it* reliably across thousands of files, ast-grep is the answer, and it is excellent for pure search too. Its one limit: it does not resolve names across files. It finds the *shape* of a call to `parseConfig`, but cannot tell you that *this* `parseConfig` is the one in `config/loader.go` and not an unrelated method of the same name two packages over.

### Graph: who actually calls, uses, or implements this?

This is the modality grep cannot fake. A code graph resolves names to declarations and records the real edges: this call site binds to *that* function; this reference is *that* symbol; this type *implements* that interface. "Who calls `parseConfig`?" stops being a text search and becomes an edge traversal — enumerate the resolved `calls` edges pointing at the one true `parseConfig` node.

The payoff is the elimination of grep's core failure mode. No comment matches, no string-literal matches, no same-name-different-symbol matches, no definition counted as a usage. You get the bound references and nothing else — the whole point of [finding usages without grep's false positives](/gortex/find-usages-without-grep-false-positives/). The same edges power [call graphs for an AI coding assistant across many languages](/gortex/call-graph-for-ai-coding-assistant/) and [graph-verified refactoring](/gortex/graph-verified-refactoring/) — safe rename, dead-code detection, cycle analysis — that text matching cannot do correctly. The trade-off: a graph needs a parse-and-resolve pipeline and an index, so it is not zero-setup the way grep is.

## What 2026 settled about agentic vs semantic search

The "just use grep" versus "use embeddings" argument has matured into something more useful than a slogan. Three data points worth holding in mind:

- **Embeddings go stale.** A widely-cited reading of Claude Code's design (as of 2026) is that it [favors agents navigating like engineers](https://wowelec.wordpress.com/2026/05/18/agentic-semantic-or-both-notes-from-the-code-search-debate/) (grep/find/read) over embedding pipelines, partly because a vector index returns functions renamed weeks ago. Disk is always current; an index is only as fresh as its last rebuild.
- **Sourcegraph dropped pure embeddings.** Cody moved to BM25F over a code graph for operational reasons (privacy, maintenance, scale) — not a rejection of ranking.
- **Lexical floor, semantic ceiling.** Turbopuffer's ContextBench reported precision ratios only — baseline Claude Code wasted roughly 1 in 3 reads, grep about 1 in 5, semantic about 1 in 8 ([no public token or latency numbers](https://www.startuphub.ai/ai-news/ai-research/2026/claude-code-benchmarking-semantic-search-vs-grep)). Semantic helps on intent queries; it is not free, and not always more precise.

The consensus is not "semantic beats grep" or "grep beats semantic." It is **hybrid**: give the agent lexical, structural, and graph search as distinct tools, let it pick per question, and let it verify against live source. The unsolved part is that almost no single tool offers all three behind one interface — agents stitch together `rg`, maybe `ast-grep`, maybe an embedding MCP that needs an API key and a vector DB.

That key-plus-DB cost is worth being fair about. [claude-context](https://github.com/zilliztech/claude-context) (MIT, ~11.7k stars as of 2026-06-06) is a strong semantic MCP — AST-aware chunking, hybrid BM25 + dense vectors, Merkle-tree incremental indexing. The fair critique is not "OpenAI-only" (it supports Ollama, VoyageAI, Gemini); it is that it *requires* an embedding provider **and** a vector database (Milvus or Zilliz Cloud) — real infrastructure for code search.

## Gortex: all three modalities behind one MCP

[Gortex](https://github.com/zzet/gortex) is a code graph and intelligence engine that indexes a repo into an in-memory graph and exposes it over CLI, an MCP server, and a web UI. The relevant point here: it maps each modality to a dedicated MCP tool, so an agent picks the right one for the question instead of grepping everything.

`search_text` is the alt-grep backbone — trigram literal/regex search that, unlike raw grep, returns the *enclosing symbol* for each match, so the agent gets a usable anchor instead of a bare line. `search_ast` runs cross-language structural patterns and ships bundled detectors — `sql-string-concat`, `weak-crypto`, `hardcoded-secret` — so an agent asks a security-shaped question without hand-writing a tree-sitter query. `search_symbols` is the ranked lexical-plus-semantic entry point (BM25, camelCase-aware, `code|docs|all`); `find_usages` and `get_callers` are the graph layer that finally answers the caller question correctly, with resolved edges and no false positives.

On Gortex's retrieval benchmark (BENCHMARK.md), the ranked lexical layer reaches **BM25 R@5 of 55.1% versus ripgrep's 17.3%** — about **3.2× ripgrep's floor** — with **R@1 of 42.3%** against ripgrep's 0.0%, and **96.8% R@5 on exact symbol-name queries**. On token efficiency, the graph tools cut a ripgrep-plus-full-read workflow from tens of thousands of tokens to hundreds — `AddObservation` drops from 31,530 to **972** tokens, with **recall@2k of 1.00 versus ripgrep's 0.00**. The headline is **3–50× fewer tokens per response** (the ~50× peak is identifier lookups, not a flat average).

Under the hood the search is the hybrid the 2026 debate landed on: BM25/FTS5 lexical, **default-on GloVe-50d embeddings** (3.8MB, embedded, zero API key — stronger MiniLM/Ollama/OpenAI opt-in), Reciprocal Rank Fusion to blend them with a per-query `alpha`, and graph-centrality reranking that floats structurally-central results up. It is Apache-2.0 and fully open-source. For how the graph itself is built and queried, see the pillar on [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/).

### Honest limits

Gortex is not the right answer for every search. Plain grep/ripgrep is still best for raw text and log scans and any unindexed corpus where you want an answer *right now* — Gortex is a binary plus a daemon, heavier to bootstrap than a single static binary, with no in-browser story. ast-grep remains the better tool for codemods and large structural rewrites, where transformation, not matching, is the goal. And Gortex's semantic recall on concept and multi-hop tiers is modest (R@5 ~25–30%); its edge is the exact tier (96.8%) and deterministic graph edges, not embedding supremacy — treat it as "hybrid lexical + structural + deterministic graph," not "best embeddings." (Benchmarks are single-machine numbers and vary 2–5× by hardware.)

## When to use which: a decision table

| Your question | Use this | Why |
|---|---|---|
| Find an exact string, error message, or scan logs | grep / ripgrep / `search_text` | Universal, fast, zero false-negative on literals |
| Search a multi-GB or multi-repo corpus by regex | Zoekt / `search_text` | Persistent trigram index, sub-50ms |
| Match a code **shape** (call form, pattern) | ast-grep / `search_ast` | AST match ignores comments, strings, whitespace |
| Run a large codemod / structural rewrite | ast-grep | Match-and-transform leader |
| Best-ranked symbol for a fuzzy name | `search_symbols` (BM25) | R@5 55.1% vs ripgrep 17.3% |
| Who calls / uses / implements this? | `find_usages`, `get_callers` | Resolved edges — zero false positives |
| Natural-language "find auth-related code" | semantic / hybrid (`search_symbols`) | Embeddings for intent, fused with lexical |

## FAQ

**Is semantic code search better than grep for AI agents?**
It depends on the question. Grep/ripgrep wins for exact strings, logs, and unindexed code with zero setup. Semantic search wins for intent queries like "find code related to authentication" on large codebases. The 2026 consensus is hybrid: give the agent lexical, structural, and graph search as tools and let it verify results against disk. Sourcegraph Cody even dropped pure embeddings for BM25 over a code graph because embeddings went stale and scaled poorly.

**What is the difference between ast-grep and grep?**
grep matches text; ast-grep matches code structure. ast-grep parses files into a tree-sitter AST and matches patterns written like real code with `$METAVAR` wildcards, so it ignores whitespace and formatting and avoids false positives in comments or strings. Use grep for raw text scans and ast-grep for codemods and lint rules where you need to match a code shape reliably.

**What's the best grep replacement for AI coding agents?**
No single lexical tool is enough — the right replacement gives the agent all three search modalities. Lexical (trigram/BM25) for text, structural (tree-sitter) for code shapes, and a resolved graph for "who actually calls this." Gortex unifies them via `search_text`, `search_ast`, and `find_usages`/`get_callers`, hitting a BM25 R@5 of 55.1% versus ripgrep's 17.3% at 3–50× fewer tokens per response.

**How does trigram code search differ from grep?**
Plain grep scans every file linearly on each query; trigram search (Zoekt, Sourcegraph) pre-builds an index of 3-character sequences with byte positions, turning a regex into index lookups — sub-50ms on multi-gigabyte codebases. ripgrep splits the difference: no persistent index, but literal extraction lets it skip most bytes before running the full regex engine.

**Why do AI agents use grep instead of semantic search?**
grep is the only universal primitive — it works on any repo and any language with no index, embedding key, or vector database. That convenience is also the trap: agents reach for grep on structural and graph questions it cannot answer, returning noisy textual matches (comments, strings, unrelated symbols) that the model then burns tokens reading and filtering.

## Bottom line

There is no single grep replacement for an AI agent, because "search" is three different questions. Lexical asks where text appears, structural asks where a code shape appears, graph asks who actually calls or uses a symbol. Grep does the first — universally, which is why agents default to it — but most code questions are the second or third, and that is where the token waste and false positives come from. Keep grep for logs and raw scans, keep ast-grep for codemods, and add a resolved graph for relational questions. Gortex puts all three behind one MCP — `search_text`, `search_ast`, `search_symbols`, and `find_usages`/`get_callers` — so the agent stops greping its way through questions grep was never meant to answer.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
