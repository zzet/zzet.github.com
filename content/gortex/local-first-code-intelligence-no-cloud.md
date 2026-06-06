---
title: "Self-Hosted Code Intelligence for AI Agents: Local-First, No Cloud"
description: "Self-hosted code intelligence for AI agents that runs offline: air-gapped, no API key, Apache 2.0. How local code graphs compare and where Gortex fits."
date: 2026-06-05
lastmod: 2026-06-06
draft: false
slug: "local-first-code-intelligence-no-cloud"
keywords: ["self-hosted code intelligence for AI agents", "local-first AI code search no cloud", "private codebase AI air-gapped", "open source code intelligence apache", "find implementations of an interface", "local code intelligence no api key"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

Your security team's answer is "no." The AI coding tool wants to upload your repository to index it, and your codebase is regulated, defense-adjacent, or just the company's core IP. So the agent runs blind — it greps, it guesses, it hallucinates functions — because the one thing that would make it accurate, a real index of your code, is the one thing you cannot ship off the machine.

This is the search behind this article: **self-hosted code intelligence for AI agents** that never leaves your hardware. Not a privacy toggle on a cloud product. A tool that can run in an environment where all network egress is blocked at the OS level, with an open license your auditors can read, and ideally one your agent can actually call over MCP.

The honest news is that this category exists and has more than one good answer. This piece maps the local-first options fairly first — including the ones that aren't Gortex — then shows where a local code graph fits and where it does not.

> **Short answer:** Several open-source tools run entirely on your machine with no API key: [codanna](https://github.com/bartolli/codanna) (Apache 2.0), [ChunkHound](https://github.com/ofriw/chunkhound) (MIT), and Gortex (Apache 2.0), all exposed to AI agents over MCP. The practical differences are startup cost and depth. codanna downloads a ~150 MB embedding model on first use; ChunkHound expects a local embedding server (Ollama, vLLM, etc.). Gortex ships a 3.8 MB GloVe model baked into the binary, so it works the instant it's installed — no model download, no database, no network — and adds a deterministic graph (calls, implements, references) on top of search, so an agent can ask "what types implement this interface?" and get type-checked answers, not vector guesses.

## What "local-first" and "air-gapped" actually mean

These terms get used loosely, so pin them down before comparing tools.

**Local-first** means indexing, search, and any inference run in-process on your hardware. Your code is read from disk, processed in memory, and the answers go back to your agent — no round-trip to a vendor's servers. A "privacy mode" checkbox on a SaaS product is not this; the bytes still traverse someone else's network.

**Air-gapped** is stricter and is the relevant baseline for regulated work. It means [zero bytes leave the machine, enforced at the network or OS level](https://www.bodegaone.ai/blog/air-gapped-ai-coding-guide-2026) — all egress blocked — not by trusting an in-app setting. For HIPAA, SOC 2, ISO 27001, or NIST 800-171 (Controlled Unclassified Information) work, that is the correct default. A tool that "doesn't store your code" still fails an air-gap test if it has to open a socket to function. The question is not "does the vendor promise privacy?" but "can this tool run with the network cable pulled out?"

That distinction is why air-gapped procurement keeps coming back to open-source, in-process tools. If the binary can't phone home, there is nothing to trust and nothing to audit at runtime.

## The cloud path is consolidating upmarket

The cheap cloud option you remember may not exist anymore. [Sourcegraph terminated Cody Free and Cody Pro](https://sourcegraph.com/blog/changes-to-cody-free-pro-and-enterprise-starter-plans): new signups stopped June 25, 2025, and all existing Free and Pro access ended July 23, 2025. Cody is now enterprise-only, bundled with Sourcegraph's enterprise platform (cited around $59/user/month, with deals in the five figures), and individuals were redirected to a separate agent product, Amp.

The picture rhymes across the cloud-first vendors:

| Tool | Posture (as of 2026) | Air-gap? | Price |
|---|---|---|---|
| [Cody / Sourcegraph](https://sourcegraph.com/blog/changes-to-cody-free-pro-and-enterprise-starter-plans) | Enterprise-only since July 2025; deepest cross-repo graph | Enterprise self-host | ~$59/user/mo, 5-figure deals |
| [Greptile](https://www.greptile.com/) | Cloud-first AI code *review*; builds a graph index; SOC 2 Type II; doesn't store code | VPC self-host (custom) | from $30/user/mo |
| [Tabnine](https://www.tabnine.com/pricing/) | Completion + agent; zero code retention; BYO-LLM | Yes, on Enterprise tier | $39–$59/user/mo; air-gap is custom |
| [Continue.dev `@codebase`](https://docs.continue.dev/reference/deprecated-codebase) | Embeddings retrieval **deprecated**; steering users to MCP servers | Local-capable | Open source |

Two of these are honest air-gap options if you can pay for them. [Tabnine genuinely supports SaaS, VPC, on-premises, or fully air-gapped deployment](https://www.tabnine.com/pricing/) with zero code retention, no training on your code, and bring-your-own-LLM — but air-gap is gated to its Enterprise tier and you run your own model server. Greptile offers VPC self-hosting under custom pricing, though it is a PR-review agent, not your coding agent's context layer.

The Continue.dev line is the telling one. Its [`@codebase` context provider is deprecated](https://docs.continue.dev/reference/deprecated-codebase); Continue now steers users toward agent-mode search tools and MCP servers. The old "embed the repo and retrieve chunks" approach is being replaced by MCP-served code intelligence — exactly the shape of the local tools below.

## The local-first menu, fairly

Strip away the cloud and you land on a small set of tools that run entirely on your machine and expose code intelligence to an agent over MCP. These are worth knowing even if you never touch Gortex.

### codanna: fast, truly offline, with a first-use model download

[codanna](https://github.com/bartolli/codanna) positions itself as a local code-intelligence MCP server and CLI — "X-ray vision for your agent." It is genuinely local-first: [Apache 2.0, runs entirely offline with no cloud or API keys](https://github.com/bartolli/codanna), tree-sitter ASTs plus AllMiniLM local embeddings, a Tantivy + mmap cache, sub-10ms lookups, roughly 75k symbols/second, around 15 languages (~687 stars as of mid-2026). It does semantic, relationship, and implementation lookup, and it is clean and quick.

One caveat, to set expectations honestly: codanna's embedding model is ~150 MB and is auto-downloaded on first use. So it is local, but not zero-download-to-start — in an air-gapped environment you stage that model first.

### ChunkHound: strong local messaging, provider-flexible, search-shaped

[ChunkHound](https://github.com/ofriw/chunkhound) has the sharpest local-first messaging in the space — "100% local," "your code stays on your machine," explicitly targeting security-sensitive and offline environments. It is MIT-licensed. To clear up a common misconception: **ChunkHound does not require an OpenAI API key.** [It runs fully offline with no key](https://github.com/ofriw/chunkhound) using a local embedding server (Ollama, vLLM, LocalAI, TEI, or any OpenAI-compatible endpoint) or its regex search; an OpenAI or VoyageAI key is optional and only needed if you deliberately choose those hosted providers.

ChunkHound's strengths are provider flexibility, multi-hop semantic search, and broad file coverage (~32 languages including JSON, YAML, Markdown, PDF). Its structural limit is the category it lives in: it is search and retrieval, not a deterministic call/implements graph. It finds chunks that are *lexically or semantically similar*; it does not resolve that type A actually satisfies interface B.

### The honest scorecard

| | codanna | ChunkHound | Gortex |
|---|---|---|---|
| **License** | Apache 2.0 | MIT | Apache 2.0 |
| **Runs offline, no API key** | Yes | Yes | Yes |
| **Zero model download to start** | No (~150 MB first use) | No (external embedding server) | Yes (3.8 MB baked) |
| **Deterministic graph** (calls/implements/refs) | Relationship lookup | Search/retrieval only | Yes |
| **Signed supply chain** | Not advertised | Not advertised | Sigstore + SLSA 3 |
| **Languages** | ~15 | ~32 | 257 |

Read it honestly: all three are real local-first tools that run with no API key. The wedge is not "we're the only local one" — that would be a strawman. It's startup cost, the kind of answer you get back, and whether a security team can verify the binary before it touches a restricted network.

## Self-hosted code intelligence for AI agents you can verify

[Gortex](https://github.com/zzet/gortex) is an Apache-2.0 code graph and intelligence engine. It indexes your repositories into an in-memory knowledge graph and exposes that graph over a CLI, an MCP server, and a web UI. For the conceptual background, start with [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/). Three properties matter for the air-gapped, no-cloud job specifically.

**Smart from second zero — no model download, no network.** Gortex is [in-process with zero external dependencies: no database, no network, no model download to start](https://github.com/zzet/gortex). Its default semantic layer is a GloVe-50d model, **3.8 MB embedded directly in the binary**. That is the precise, fair contrast with the peers above: where codanna pulls ~150 MB on first use and ChunkHound expects you to stand up an embedding server, Gortex's index and search work the moment the binary lands. For an air-gapped install, "nothing to download" removes a whole staging step.

**A deterministic graph, not vector guesses.** This is the part search-only tools structurally cannot do. Take the query behind a lot of this traffic — *find implementations of an interface*. Grep can only match text, so it misses implementers that never mention the interface name and produces false positives from comments and strings. [LSP-style go-to-implementation resolves type-checked interface satisfaction](https://github.com/blackwell-systems/agent-lsp) instead. Gortex exposes that as a tool:

```bash
# Which concrete types actually satisfy this interface?
gortex query implementations Storage         # type-checked, not a name match

# Then navigate the result deterministically
gortex query usages BlobStore.Put            # every reference, typed by context
gortex query deps BlobStore                  # what this symbol depends on
```

`find_implementations` returns the actual types that satisfy a given interface, `find_usages` returns every reference (each tagged with a typed context — parameter type, return type, field, call — so no false positives from comments or strings), and `get_dependencies` returns what a symbol depends on. All of it runs locally with no cloud call. This is "graph **and** vector," not "graph versus vector": Gortex fuses BM25, vectors, and RRF for ranking, then the graph supplies the *deterministic* edges embeddings only approximate. The fuller argument is in [the code-intelligence MCP server your setup is missing](/gortex/mcp-server-for-codebase-context/).

**A supply chain you can attest before it crosses the air gap.** Importing any binary into a restricted network is a procurement event. Gortex releases are [Sigstore-signed and built to SLSA Level 3](https://slsa.dev/) — the highest of SLSA's build levels, requiring an isolated build environment and provenance generated in a trusted control plane that users can't falsify. A security team can verify the artifact's provenance before it touches the offline network. The local peers are genuinely local but don't advertise this attestation; for a regulated import, that is the difference between "trust the README" and "check the signature."

A first install on an offline workstation:

```bash
curl -fsSL https://get.gortex.dev | sh      # one signed binary; verify the signature first
gortex track ./your-repo                     # indexes locally, in-memory
gortex daemon start --detach                 # detected agents wired to the MCP server
# no DB, no model pull, no egress — pull the network cable and it still works
```

On cost, the math is simple: Apache 2.0, no per-seat fee, versus $30–$59/user/month cloud tiers. On tokens, reading a symbol via `get_symbol_source` returns ~80% fewer tokens than reading the whole file, and across BENCHMARK.md's query set Gortex lands **3–50× fewer tokens per response** versus ripgrep-plus-full-read (peak ~50× on identifier lookups), at median recall@2k of 1.00 versus 0.00 for ripgrep.

### Where Gortex is the wrong tool

The house style is to tell you where this loses, so the rest is credible.

- **Cloud tools win on hosted convenience.** Cody (enterprise), Greptile, and Tabnine SaaS give you team dashboards, IP indemnification, dedicated compliance support, and zero local resource use. If you want someone else to run and scale the index, that is a real advantage local-first does not offer.
- **Tabnine can also go air-gapped** — at its Enterprise price and with you running a model server. It is a fair paid alternative, not a tool that can't go local.
- **It is a binary + daemon, not a browser tool.** Heavier to bootstrap than a WASM/in-browser option; there is no in-browser story.
- **Some refactors are Go-only for now.** `move_symbol` and `inline_symbol` are Go-only at present; `rename_symbol` (cross-file) and `safe_delete_symbol` are broader.
- **Default embeddings are lightweight.** The baked GloVe-50d model is zero-setup with no API key; for stronger semantic recall you opt into MiniLM, Ollama, or OpenAI. Gortex's edge is hybrid retrieval plus a *deterministic* graph — exact-symbol recall is where it dominates (bm25 R@5 = 96.8%), while concept and multi-hop recall are modest.

If you want the direct head-to-head against the other small local tools, see [local code-graph MCP servers compared: codanna, ChunkHound, Serena, Context7](/gortex/local-code-graph-mcp-servers-compared/). If you came here from the enterprise side, the [self-hosted Sourcegraph and Cody alternative](/gortex/sourcegraph-cody-alternative-self-hosted/) covers that comparison.

## FAQ

### What is the best self-hosted code intelligence for AI agents that runs with no API key?

Several open-source tools run entirely on your machine with no API key: codanna (Apache 2.0), ChunkHound (MIT), and Gortex (Apache 2.0). All three expose code intelligence to AI agents over MCP and work offline. The practical difference is startup and depth: codanna downloads a ~150 MB embedding model on first use and ChunkHound expects a local embedding server, while Gortex ships a 3.8 MB GloVe model baked into the binary, so it works the instant it's installed — no model download, no database, no network. Gortex also adds a deterministic graph (calls, implements, references) on top of search, so an agent can ask "what types implement this interface?" and get exact answers, not vector guesses.

### Can I use AI code intelligence in an air-gapped or offline environment?

Yes. Air-gapped means all network egress is blocked at the OS level so the tool cannot phone home. Local-first tools like Gortex, codanna, and ChunkHound run fully offline because indexing, search, and the graph all execute in-process on your hardware. Gortex in particular has zero external dependencies — no database, no network, no model download to start — and ships Sigstore-signed, SLSA Level 3 binaries, so a security team can verify the supply chain before importing it into a restricted network. Tabnine can also be deployed fully air-gapped, but that requires its Enterprise tier and running your own LLM server.

### Does ChunkHound require an OpenAI API key?

No. As of 2026, ChunkHound is MIT-licensed and local-first. It runs fully offline with no API key when you use a local embedding server (Ollama, vLLM, LocalAI, TEI, or any OpenAI-compatible endpoint) or its regex search. An OpenAI or VoyageAI key is optional and only needed if you specifically choose those hosted embedding providers. Its own site emphasizes "100% local" and "your code stays on your machine."

### Is Cody (Sourcegraph) still free?

No. Sourcegraph stopped new Cody Free and Cody Pro signups on June 25, 2025 and terminated all existing Free and Pro access on July 23, 2025. Cody is now enterprise-only, bundled with Sourcegraph's enterprise platform (cited around $59/user/month, with five-figure enterprise deals), and individual users were redirected to Sourcegraph's new agent product, Amp. If you want free, private code intelligence today, a self-hosted open-source tool like Gortex, codanna, or ChunkHound is the path.

### How do I find all implementations of an interface without grep?

Grep can only match text, so it misses implementations that don't mention the interface name and produces false positives. A code-intelligence graph resolves type relationships instead. In Gortex, the `find_implementations` tool returns the actual types that satisfy a given interface (type-checked interface satisfaction), and it runs locally with no cloud call. Pair it with `find_usages` (every reference to a symbol) and `get_dependencies` to navigate the codebase deterministically inside your AI agent.

## Bottom line

The cloud path is consolidating upmarket — Cody killed its free and pro tiers, and the air-gap-capable vendors (Tabnine, Greptile VPC) gate it behind enterprise pricing and a model server you operate. The local-first answer is real and has more than one good option: codanna and ChunkHound are genuinely offline, no-key tools, and you should pick by job. Gortex's specific edges for the regulated, air-gapped case are that it works from second zero with nothing to download (3.8 MB baked GloVe), fuses graph and vectors — hybrid BM25 + vector + RRF ranking with a deterministic graph adding type-checked edges (`find_implementations`, `find_usages`, `get_dependencies`) that embeddings only approximate — and ships a Sigstore-signed, SLSA 3 supply chain a security team can verify before it crosses the air gap. Apache 2.0, no per-seat, runs with the network cable pulled.

If that is your situation, it costs one command to try: [github.com/zzet/gortex](https://github.com/zzet/gortex).
