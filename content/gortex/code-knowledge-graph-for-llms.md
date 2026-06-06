---
title: "Code Knowledge Graph for LLM Agents: What It Is and Why You Need One"
description: "A code knowledge graph for LLM agents: a crisp definition of nodes and edges, graph vs grep vs vector, and how to index a repository into a graph."
date: 2026-05-26
lastmod: 2026-06-06
draft: false
slug: "code-knowledge-graph-for-llms"
keywords: ["code knowledge graph for LLM", "what is a code knowledge graph", "knowledge graph of your codebase", "index a repository into a graph", "code property graph explained", "repo map for LLM", "code graph for ai agents"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

Your agent is asked one ordinary question: *what breaks if I change this function?* With grep, it greps the name, gets a wall of text matches — some in comments, some in strings, some a different symbol with the same name — and starts guessing. With a vector index, it gets the chunks that *look* most similar, which is not the same as the chunks that actually call the function. Both answers are approximations. Neither knows, for certain, who the real callers are.

A **code knowledge graph for LLM** agents fixes the certainty problem. Instead of matching text or measuring similarity, it stores the *resolved relationships* in your codebase as a queryable graph — and then "who calls this?" is a traversal, not a guess.

{{< lead >}}A code knowledge graph is a typed, queryable graph of your codebase: the nodes are files, symbols (functions, classes, types), tests, and infra (routes, env vars, models); the edges are the real relationships between them — calls, imports, implements/overrides, references, and dataflow. Instead of an LLM guessing how code connects by reading text, it traverses verified edges, so retrieval is deterministic and grounded in structure rather than similarity.{{< /lead >}}

This is the pillar for everything else on this blog about giving AI agents real code intelligence. It defines the category, walks the build pipeline, sets the graph-vs-grep-vs-vector mental model straight, and links down to the specific problems — [finding usages without grep's false positives](/gortex/find-usages-without-grep-false-positives/), [the token waste of code retrieval](/gortex/ai-agent-token-waste-code-retrieval/), [impact analysis](/gortex/impact-analysis-what-breaks-if-i-change-this/) — that a graph solves. If you read nothing else, read the definition above and the comparison table below.

---

## What is a code knowledge graph: the full taxonomy

The one-sentence definition is above. Here is the precise version — the node and edge taxonomy that distinguishes a real code graph from a glorified symbol list.

**Nodes** are the things in your codebase:

| Node kind | Examples |
|---|---|
| **Files / packages** | source files, modules, directories |
| **Symbols** | functions, methods, classes, structs, interfaces, fields, constants |
| **Types** | the type each symbol declares or returns |
| **Tests** | test functions and the symbols they exercise |
| **Infra** | HTTP routes, env vars, data models, message topics, k8s resources |

**Edges** are the real relationships between those nodes:

| Edge kind | Meaning |
|---|---|
| **calls** | A invokes B |
| **imports** | this file/package depends on that one |
| **implements / overrides** | this type satisfies that interface; this method overrides that one |
| **references** | this site reads or mentions that symbol |
| **dataflow** | a value moves from here to there — `value_flow`, `arg_of`, `returns_to` |
| **cross-repo contracts** | a provider in repo A satisfies a consumer in repo B, normalized to a canonical ID like `http::GET::/api/users/{id}` |

That last two rows are what separate a serious code graph from a toy. **Dataflow edges** let you ask "where does this tainted input end up?" — a question grep and embeddings cannot answer at all. **Cross-repo contract edges** let the graph span a polyrepo or a service mesh: an HTTP route declared in one service is matched to its caller in another, so an agent can see that deleting an endpoint orphans a consumer three repos away.

The practical test for whether something is a code knowledge graph: can you traverse it? If the answer to "who calls this?" is a graph walk that returns resolved callers, it's a graph. If it's a text search or a similarity ranking, it isn't — it's a useful index that resembles one.

### The ancestry: code property graphs

The idea isn't new. The **code property graph (CPG)** was introduced by Yamaguchi, Golde, Arp, and Rieck at IEEE S&P 2014 and first implemented in [Joern](https://docs.joern.io/code-property-graph/). A CPG merges three classic program representations — the abstract syntax tree (AST), the control-flow graph (CFG), and the program dependence graph (PDG) — into one directed, edge-labeled, attributed graph. Nodes are program constructs with key-value attributes (`METHOD`, `LOCAL`, …); edges are labeled relations (`CONTAINS`, control-flow, dataflow). See the [code property graph definition and origin](https://en.wikipedia.org/wiki/Code_property_graph) for the rigorous version.

A CPG was built for security analysis — find vulnerable patterns by querying structure and dataflow together. A *code knowledge graph for LLM agents* is the agent-era, retrieval-tuned descendant: same backbone (AST + resolved edges + dataflow), but optimized for the questions an agent asks during a task ("show me the real callers," "give me the minimal context to edit this," "what's the blast radius?") and exposed over an interface an agent can call.

---

## Why your agent needs a code knowledge graph for LLM reasoning

An agent without a graph has two retrieval tools: text search and similarity search. Both are approximations, and approximation is exactly where agents waste tokens and ship bugs. A graph buys four concrete things, each measurable.

**1. Deterministic retrieval — real edges, not guesses.** When the graph says A calls B, it's because the resolver built a `calls` edge from the AST and resolved the name to its actual declaration. There's no "probably." Grep returns lines that contain a string; a vector index returns chunks ranked by cosine similarity. A graph returns the resolved relationship. That's the difference between an answer and a lead. This is the whole reason an agent can [find usages without grep's false positives](/gortex/find-usages-without-grep-false-positives/).

**2. Blast radius — impact is a traversal, not a search.** "What breaks if I change this?" is the question that ungrounded agents answer worst, because text search can't follow the dependency chain. On a graph it's a reverse traversal over `calls`, `implements`, and `references` edges. Done right, it's precomputed and cheap enough to run on every edit — the full mechanics are in [impact analysis: what breaks if I change this function?](/gortex/impact-analysis-what-breaks-if-i-change-this/).

**3. Token efficiency — answer from edges, not from reading files.** A practitioner benchmark of a tree-sitter/SQLite code graph reported roughly **121× fewer tokens** across representative queries — answering "what calls `ProcessOrder`?" in about 200 tokens versus about 45,000 spent reading files ([source](https://dev.to/deusdata/how-i-cut-my-ai-coding-agents-token-usage-by-120x-with-a-code-knowledge-graph-4a3d); that number is *that tool's* benchmark, not Gortex's). The general principle holds across implementations: reading file edges beats reading file bytes. This is the core argument that [your AI agent wastes most of its tokens finding code](/gortex/ai-agent-token-waste-code-retrieval/).

**4. Grounding that cuts hallucination.** Generation anchored to the repo's real structure produces code that exists. Knowledge-graph-based repository-level code generation [outperformed baseline RAG](https://arxiv.org/html/2505.14394v1): Pass@1 of **36.36%** (Claude 3.5 Sonnet via KG retrieval) versus **20.73%** for the best baseline (Local File Infilling) — a ~76% relative improvement — by grounding in real dependency structure rather than superficial similarity.

| The agent's question | grep answers with… | a graph answers with… |
|---|---|---|
| Who calls `X`? | lines containing `X` (comments, strings, same-name) | resolved `calls` edges |
| What breaks if I change `X`? | a text search you re-run by hand | a reverse traversal (blast radius) |
| What's the minimal context to edit `X`? | "read these files and hope" | the symbol + signature + callers + callees |
| Where does this value flow? | nothing | `value_flow` / `arg_of` / `returns_to` paths |

---

## How to index a repository into a graph

A code graph isn't magic; it's a pipeline. Understanding the four stages tells you why some "graphs" are better than others — and why the resolution step is the one that matters most.

```text
source files
   │  1. parse
   ▼
tree-sitter AST  ──2. extract──►  nodes + candidate edges
                                        │  3. RESOLVE names → real declarations
                                        ▼
                                  resolved graph
                                        │  4. index for search + traversal
                                        ▼
                                  queryable code knowledge graph
                                        │  keep fresh: patch on file change
                                        ▼
                                  incremental re-index
```

**1. Parse to an AST.** [tree-sitter](https://github.com/tree-sitter/tree-sitter) is the de-facto choice: an incremental, error-tolerant parser with grammars for 100+ languages that builds a concrete syntax tree and updates it cheaply as source changes — which is exactly what makes incremental re-indexing affordable later.

**2. Extract nodes and candidate edges.** Walk the AST to pull out definitions (functions, classes, types), call sites, imports, and references. At this stage a call site is just "a name being invoked" — a *candidate* edge, not a resolved one.

**3. Resolve names to real declarations.** This is the step that separates a code graph from grep. The resolver figures out *which* `process` a given call site means — across files, through imports, through interfaces — and connects the candidate edge to the actual declaration. **This resolution is what kills grep's false positives.** Without it you have a fast text index with extra steps; with it you have deterministic edges.

**4. Index for search and traversal.** Build the lexical and (optionally) semantic indexes for search, plus the adjacency structure for graph walks. Then keep it fresh: when a file changes, patch the affected part of the graph incrementally instead of re-indexing the whole repo. A stale graph grounds an agent in last week's code — which, as covered in the [hallucination article](/gortex/stop-ai-agent-hallucinating-code/), is its own failure mode.

---

## Graph vs grep vs vector: the mental model

The most common mistake is to frame this as "graph *instead of* vectors." That's wrong. The strongest systems are **graph + vector**, because the three approaches answer different questions. Here's the honest comparison — including where embeddings genuinely win.

| | grep / ripgrep | vector / embeddings | code knowledge graph |
|---|---|---|---|
| **Matches** | characters on a line | semantic similarity | resolved relationships |
| **"Who calls X?"** | noisy (false positives) | approximate | exact (`calls` edges) |
| **"Find code *like* this"** | poor | strong | weak alone |
| **Cross-file / cross-repo** | no | weak | yes |
| **Deterministic** | yes (but wrong question) | no | yes |
| **Setup cost** | none | embeddings index | parse + resolve + index |

Embeddings are good at "find me code that's *about* authentication" — fuzzy, intent-level queries where there's no exact symbol to anchor on. A graph is good at "find the *exact* callers of `AuthMiddleware`" — structural queries where similarity is beside the point. Lexical search (BM25) sits between them, dominating exact identifier lookups. A system that fuses all three — lexical for identifiers, semantic for intent, graph for structure — beats any one alone. The full treatment is in [graph RAG vs vector RAG for code](/gortex/graph-rag-vs-vector-rag-for-code/); the short version is that the graph is what makes retrieval *verifiable*, and the vectors are what make it *flexible*.

### The depth ladder: L1 symbol graph → L2 program graph → L3 system graph

Not all "code graphs" go the same depth. There's a ladder, and agents working on real systems need the top rung.

| Level | What it captures | Example | What it answers |
|---|---|---|---|
| **L1 — symbol graph** | definitions + references, per language | LSIF / [SCIP](https://sourcegraph.com/blog/announcing-scip) | go-to-definition, find-references |
| **L2 — program graph + dataflow** | calls, implements, plus dataflow | CPG / [Joern](https://docs.joern.io/code-property-graph/) | "where does this value flow?", taint paths |
| **L3 — system graph + contracts** | L2 + cross-repo contracts, infra | full code knowledge graph | "what breaks across all repos if I delete this endpoint?" |

SCIP, Sourcegraph's open successor to LSIF, is an excellent **L1** index — precise go-to-definition and find-references at multi-repo scale via a typed, human-readable symbol format, and it does that job well. But an L1 symbol graph isn't an agent-facing program graph: it has no dataflow and no impact analysis. An agent fixing a bug across a service boundary needs **L3** — and it needs it fresh, because a system graph that lags behind the code grounds the agent in a system that no longer exists. The compounding cost of getting this wrong is why [coding agents fail past 400k lines](/gortex/why-ai-coding-agents-fail-large-codebases/).

---

## You don't need to build and host a graph database

Here's the trap when people hear "knowledge graph": they think they need to stand up a graph database. You don't.

[FalkorDB](https://github.com/FalkorDB/code-graph) and [Memgraph](https://memgraph.com/graphrag) are genuinely good graph databases. As of 2026, FalkorDB uses GraphBLAS sparse-matrix math and advertises large speedups on multi-hop queries, and ships a web visualization; Memgraph offers in-memory traversals, vector search, and Cypher pipelines. Both are excellent at being a graph DB.

But a graph DB is an *empty database*, not a code graph. To turn one into a code knowledge graph you still have to:

- write the parser (AST extraction per language),
- resolve the edges (the hard part — names to declarations),
- design the schema,
- host the server (Docker / cloud), and
- keep it fresh as code changes.

That's the whole pipeline above, plus operating a database. It's also worth noting the honest limits of the code-specific demos on top of these DBs (all as of 2026 — verify current details against each project, since licenses and language coverage shift): FalkorDB's `code-graph` demo focuses on a small set of languages (Python, Java, C#) rather than broad coverage, and the demo's permissive license differs from the underlying FalkorDB database, which ships under a source-available license. Memgraph's GraphRAG is a general-purpose knowledge-graph platform, not a code-specific tool, so you build and host the code graph and write the Cypher yourself.

The one-line takeaway: **you do not need to build and host a graph database to give your AI agent a code knowledge graph.** A code-specific engine ships the parser, the resolution, the freshness, the retrieval, and the agent interface as one thing.

---

## Gortex as a live example of a code knowledge graph

[Gortex](https://github.com/zzet/gortex) is one such engine. It indexes a repository into an in-memory code knowledge graph, in-process, with no external database to operate, and exposes it over CLI, an MCP server, and HTTP. It's the finished, code-specific version of the pipeline above — and a useful concrete example of what the numbers look like at scale.

```bash
# index a repo
gortex index .

# ask the graph who calls a function — a traversal, not a grep
gortex query callers AuthMiddleware
```

```jsonc
// what your agent calls over MCP
get_callers(symbol: "AuthMiddleware")   // reverse traversal over calls edges
find_usages(symbol: "AuthMiddleware")   // every resolved reference, typed
explain_change_impact(symbol: "AuthMiddleware")  // risk-tiered blast radius
```

The scale numbers come from the project's README and published benchmarks:

| Repo | Files | Nodes | Edges | Index time |
|---|---|---|---|---|
| torvalds/linux | 70,333 | **1,690,174** | 6,239,570 | **~3 min** |
| microsoft/vscode | 10,762 | 204,501 | 808,902 | ~1 min |
| zzet/gortex (self) | 430 | 5,583 | 53,830 | 3.4s |

The Linux kernel — 70,333 files — becomes a graph of **1,690,174 nodes and 6,239,570 edges in about 3 minutes** (Apple Silicon, single machine; absolute timings vary 2–5× by hardware). It covers **257 languages** across three tiers, exposes **100+ MCP tools**, and is **Apache 2.0** — free for any use, commercial included.

On retrieval, Gortex is **hybrid, not graph-only**. It fuses BM25 lexical search, semantic embeddings, and graph-centrality reranking via Reciprocal Rank Fusion — the graph gives deterministic, verifiable edges (calls, implements, references) that embeddings only approximate, while the embeddings keep intent-level queries flexible. It's "graph + vector," exactly as the mental-model section argued. In the project's benchmarks that combination returns answers in **3–50× fewer tokens** than naive file reads (the ~50× peak is on identifier lookups, not a flat figure), and `get_symbol_source` returns a single function with about **80% fewer tokens** than reading the whole file. Impact analysis runs against a precomputed depth-3 reach index at a measured **p95 of 0.01ms**, which is what makes "what breaks?" cheap enough to ask on every edit.

For cross-repo work, Gortex auto-detects API contracts — HTTP routes, gRPC, GraphQL, message topics, env vars — and normalizes them to canonical IDs (`http::GET::/api/users/{id}`), then matches providers to consumers across repos and flags orphans. That's the **L3 system graph** rung, with the freshness handled by fsnotify-driven incremental patches (~200ms re-index vs 3–5s full) rather than full re-indexes. To put the graph in front of an agent, the same engine doubles as [the code-intelligence MCP server your setup is missing](/gortex/mcp-server-for-codebase-context/).

### Honest limits

A code graph isn't a free win, and Gortex isn't an exception:

- It's a binary plus a daemon — heavier to bootstrap than a browser/WASM tool, and there's no in-browser story. For a one-off literal search in an unindexed directory, grep is still the faster reach.
- The default semantic embeddings are lightweight **GloVe-50d** (zero setup, no API key); for stronger semantic recall you opt into MiniLM, Ollama, or OpenAI.
- `move_symbol` and `inline_symbol` are **Go-only for now**. Cross-language refactor parity isn't there yet.
- Benchmarks are single-operator-machine numbers; they're reproducible (`gortex bench …`) but absolute timings vary by hardware. Cite the **3–50× token range**, not a flat 50×.
- A graph reduces hallucination and false positives; it does not abolish the model's capacity to misuse a *real* symbol. Ground, then verify.

None of that changes the core claim: for the structural questions an agent asks during a task, deterministic graph edges beat a text match or a similarity score.

---

## FAQ

### What is a code knowledge graph?

A code knowledge graph is a typed, queryable graph of your codebase: the nodes are files, symbols (functions, classes, types), tests, and infra (routes, env vars, models), and the edges are the real relationships between them — calls, imports, implements/overrides, references, and dataflow. Instead of an LLM guessing how code connects by reading text, it traverses verified edges, so retrieval is deterministic and grounded in structure rather than similarity.

### How is a code knowledge graph different from a vector or embeddings index?

A vector index finds code that is semantically similar; a knowledge graph follows exact, resolved relationships. Embeddings approximate "related"; a graph knows that A calls B because the edge was resolved from the AST. The strongest setups are hybrid, blending BM25 lexical search, embeddings, and graph structure — so it's "graph + vector," not "graph versus vector." The graph is what makes retrieval verifiable.

### Why does an AI coding agent need a code knowledge graph?

Four reasons. Deterministic retrieval: it returns real edges instead of guesses. Blast radius: "what breaks if I change this?" becomes a graph traversal, not a text search. Token efficiency: answering "what calls X?" can cost roughly 200 tokens instead of tens of thousands spent reading files. And grounding that cuts hallucination, because generated code is anchored to the repository's real structure — one study measured Pass@1 of 36.36% with a knowledge graph versus 20.73% for baseline RAG.

### How do you index a repository into a graph?

Parse every file with tree-sitter into an AST, extract definitions, call sites, imports, and references, then resolve names to their actual declarations across files (this resolution step is what removes grep's false positives). Index the result for both search and traversal, then keep it fresh by patching the graph incrementally whenever a file changes instead of re-indexing the whole repo.

### Do I need to run a graph database like FalkorDB or Memgraph to give my agent a code knowledge graph?

No. FalkorDB and Memgraph are excellent general-purpose graph databases, but they hand you an empty database — you still have to build the parser, resolve the edges, design the schema, host the server, and keep it fresh. A code-specific engine such as Gortex ships all of that and runs in-process with no database to operate, indexing 1,690,174 nodes from the Linux kernel in about 3 minutes across 257 languages and exposing the graph over 100+ MCP tools under Apache 2.0.

---

## Bottom line

A code knowledge graph is the difference between an agent that guesses how your code connects and one that knows. Nodes are files, symbols, types, tests, and infra; edges are calls, imports, implements, references, dataflow, and cross-repo contracts. It descends from the code property graph but is tuned for the questions agents actually ask, and it earns its place with four measurable wins: deterministic retrieval, blast radius, token efficiency, and grounding that cuts hallucination.

The honest framing is "graph + vector," not "graph vs vector" — the graph gives verifiable edges, the embeddings give flexibility, and lexical search anchors the identifiers. And you don't need to stand up a graph database to get one: a code-specific engine ships the parser, resolution, freshness, and agent interface as a single thing. Gortex is one example — 1.69M nodes from the Linux kernel in ~3 minutes, 257 languages, 100+ MCP tools, hybrid retrieval, Apache 2.0, in-process with no DB. Whatever you adopt, give your agent edges instead of guesses.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
