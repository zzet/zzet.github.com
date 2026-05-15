---
title: "From GitNexus to Gortex: Building a Go Code Intelligence Engine"
description: "How I experimented with GitNexus, discovered its limitations, and built Gortex — a Go code intelligence engine with 48 MCP tools, 33-language support, inline symbol editing, API contract detection, multi-repo mode, and measurable 94%+ token reduction for AI agents."
date: 2026-04-08
lastmod: 2026-05-15
draft: false
slug: "from-gitnexus-to-gortex"
keywords: ["gortex", "gitnexus alternative", "code intelligence mcp", "golang mcp server", "code knowledge graph", "ai coding agent context", "mcp tools codebase", "gortex vs gitnexus", "multi repo mcp", "multi repository code intelligence", "semantic search codebase", "api contract detection mcp", "edit symbol mcp", "gitnexus", "git nexus", "gitnexus github"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

AI coding agents waste a lot of tokens reading files. You ask for a refactor, the agent reads the file, then the imports, then the dependencies, then the files *those* import. By the time it gets to your question it has burned through a big chunk of the context window just mapping out what should have taken two seconds.

That's what pushed me toward GitNexus. And then toward building Gortex.

## GitNexus

GitNexus, by [Abhigyan Patwari](https://github.com/abhigyanpatwari/GitNexus), pre-indexes a repository into a knowledge graph. The agent queries the graph instead of crawling the filesystem. It ships with a CLI, an MCP server, a browser graph explorer at gitnexus.vercel.app, and 16 MCP tools — hybrid search (BM25 + semantic), blast radius analysis, Cypher query support.

I used it daily for a few weeks. The graph approach works: "which services call this function?" goes from a recursive file traversal to one MCP call. The browser explorer is nice — a force-directed graph you can drag around and inspect.

## What didn't work for me

**Language coverage.** GitNexus supports eight languages: JavaScript, TypeScript, Python, Java, Go, Rust, C++, C#. My day-to-day has Go services, Ruby tooling, Bash scripts, Protobuf definitions, Dockerfiles, HCL. The graph was silent on all of that.

**Too few tools.** Sixteen tools covers the basics, but I kept needing things they didn't expose: dead code detection, architectural constraint checks, execution flow tracing, interface implementation lookup. Every time I hit one of those gaps I fell back to reading files.

**Stale graph.** The persistent LadybugDB storage survives restarts, which is useful. But it can quietly go out of date, and there wasn't a clean way to know when it had.

**TypeScript startup time.** Not a language complaint — TypeScript is fine. But I was running this as a background watcher process across different repos all day. A Go binary starts faster and uses less memory.

## The Go prototype

I started a weekend prototype with one goal: replicate the graph-query core and see if the architecture felt better. Go has tree-sitter bindings, cobra for CLI, fsnotify for watching files. A day later it was working well enough to keep going.

After a few evenings and weekends: **Gortex** — a code intelligence engine that builds a knowledge graph from your repositories and exposes it over CLI, MCP server, and a web UI.

## What Gortex does differently

### 33 languages

Code: Go, TypeScript, JavaScript, Python, Rust, Java, C#, Kotlin, Swift, Scala, PHP, Ruby, Elixir, C, C++, Dart, OCaml, Lua, Zig, Haskell, Clojure, Erlang, R, Bash/Zsh. Config and data: SQL, Protobuf, Markdown, HTML, CSS, YAML, TOML, HCL/Terraform, Dockerfile.

The functional languages (OCaml, Haskell, Clojure, Zig) matter — they're usually the ones that get left out.

### 48 MCP tools

The 48 tools fall into nine groups: core navigation, graph traversal, coding workflow, agent-optimized calls, analysis, safety, quality, code generation, and multi-repo management.

The key tool is `smart_context`. One call returns a symbol's definition, callers, callees, type relationships, and community membership. That replaces 5–10 file reads. Gortex now measures this per call — every source-reading tool returns a `tokens_saved` field, and `graph_stats` gives you session totals: tokens sent, tokens saved, and an efficiency ratio. On my codebases it's usually 20–25x.

### Interface inference

Gortex builds IMPLEMENTS edges for Go, TypeScript, Java, Rust, C#, Scala, Swift, and Protobuf. The agent can find all implementations of an interface in one call without reading every file.

### Semantic search

BM25 + vector search fused with Reciprocal Rank Fusion (RRF). Built-in GloVe word vectors work offline with no setup. For better quality, point `--embeddings-url` at a local Ollama or an OpenAI-compatible endpoint. Offline transformer backends are available as build tags (ONNX, GoMLX, Hugot).

### Disk persistence, fast restart

Early versions re-indexed everything on every start. On large repos that was slow. Now Gortex saves the graph to disk on shutdown and restores it on startup, then re-indexes only the files that changed. Cold start on a medium Go codebase: ~200ms instead of 3–5 seconds. The cache is keyed by repo path + git commit hash, so it invalidates automatically on upgrade.

### Watch mode

The graph updates as files change. fsnotify with debouncing, per-file patches — no full rebuild. The agent always has a current graph.

### Guard rules

You define which dependency patterns are allowed and which are forbidden in `.gortex.yaml`. Gortex checks every proposed change against them and flags violations before the code hits review. Useful when an agent tries to import infrastructure code into a domain package.

### Edit and rename by symbol ID

`edit_symbol` takes a symbol ID and a new body, patches the file, and re-indexes the symbol. `rename_symbol` updates every reference across the codebase. The agent doesn't need to read the file first — the graph knows where the symbol is. The typical "read file → find the spot → write the whole file back" sequence becomes one MCP call.

### Contract detection across repos

`get_contracts` finds HTTP routes (from framework annotations in gin, Express, FastAPI, Spring, etc.), gRPC service definitions, GraphQL schemas, message topics (Kafka, NATS, RabbitMQ), WebSocket events, and env vars. `check_contracts` finds providers with no consumers and consumers with no matching provider. Contracts are normalized to IDs like `http::GET::/api/users/{id}`, so a Go server and a TypeScript client match correctly even though they're in different repos. The kind of mismatch this catches usually only shows up at runtime.

### MCP prompts

Three prompts for common workflows:

- `orientation` — graph stats, community clusters, execution flows, and key symbols for an unfamiliar codebase
- `pre_commit` — affected symbols, blast radius, risk level, and which tests to run before committing
- `safe_to_change` — blast radius, edit plan, affected tests, and a risk assessment for a specific symbol

### Community skills

`gortex skills` clusters the codebase into functional communities and writes a `SKILL.md` for each one. Claude Code picks these up automatically. Each file has the key files, entry points, cross-community connections, and ready-to-use MCP calls for that part of the codebase.

### Nine IDE integrations

`gortex init` detects which editors are installed and sets each one up:

- **Claude Code** — `.mcp.json`, slash commands, global skills, `PreToolUse` hook, `CLAUDE.md`
- **Kiro** — MCP config, steering files, three agent hooks (smart-context assembly, post-edit blast radius, pre-read enrichment)
- **Cursor** — `.cursor/mcp.json` (committable, so the whole team gets it)
- **VS Code / GitHub Copilot** — `.vscode/mcp.json` for Copilot agent mode
- **Windsurf** — global `mcp_config.json`, merged without overwriting existing servers
- **Continue.dev** — `.continue/mcpServers/gortex.json`
- **Cline** — writes to the extension globalStorage with an `alwaysAllow` list
- **OpenCode** — `.opencode/config.json`
- **Antigravity** — a knowledge item that tells it to use Gortex CLI queries instead of grep

Run once, every editor on the team is connected to the same 48 tools.

### Multi-repo

Gortex can index multiple repos into one graph. You define named projects in `~/.config/gortex/config.yaml` — each project is a list of repo paths. Per-repo settings go in `.gortex.yaml`.

```bash
# Track multiple repos into one graph
gortex track /code/api /code/frontend /code/protos

# Use a named project (defined in ~/.config/gortex/config.yaml)
gortex daemon start --project my-saas

# Check what's indexed
gortex status
```

Symbol IDs in multi-repo mode become `<repo_prefix>/<path>::<Symbol>`. The `CrossRepoResolver` links call edges across repos. Blast radius and impact analysis follow those edges and group results by repo — so "what breaks if I change this interface?" covers every tracked repo in one call.

Repos shared across projects — shared libraries, utility packages — are indexed once and referenced from multiple project graphs.

## GitNexus vs Gortex

GitNexus is a solid project. Abhigyan built the conceptual foundation for what this category of tool looks like. I built Gortex because I had needs it didn't cover, not because it's bad.

| | GitNexus | Gortex |
|---|---|---|
| Language | TypeScript | Go |
| Languages parsed | 8 | 33 |
| MCP tools | 16 | 48 |
| MCP resources | — | 7 |
| MCP prompts | — | 3 |
| Storage | Persistent (LadybugDB) | In-memory + disk snapshot |
| Watch mode | — | Yes |
| Multi-repo | — | Yes |
| Semantic search | BM25 + semantic | BM25 + vector + RRF |
| Token savings tracking | — | Per-call + session |
| Inline symbol editing | — | `edit_symbol`, `rename_symbol` |
| Contract detection | — | HTTP, gRPC, GraphQL, pub/sub, env |
| Interface inference | — | Yes (8 langs) |
| Guard rules | — | Yes |
| Community skills | — | `gortex skills` |
| IDE integrations | — | 9 (`gortex init`) |
| External services | None | None |

GitNexus advantage: the hosted explorer at gitnexus.vercel.app lets you share a codebase graph with someone who hasn't installed anything. If you're working in the eight supported languages and want a managed option, it's worth looking at.

Gortex advantage: 33 languages including functional ones, 48 tools with inline editing and contract detection, disk persistence, measurable token savings per call, nine IDE integrations from one command. If you're on a polyglot stack or working across multiple repos, Gortex covers more ground.

## Try it

```bash
# Install (macOS/Linux one-liner)
curl -fsSL https://get.gortex.dev | sh

# Homebrew alternative
brew install zzet/tap/gortex

# Once per machine — installs shell completions, hooks, editor stubs
gortex install

# Start the daemon (survives editor restarts; auto-start at login optional)
gortex daemon start --detach
gortex daemon install-service  # optional: auto-start at login

# Track repos
gortex track ~/projects/myapp
gortex track ~/projects/frontend ~/projects/protos  # multi-repo in one graph

# Per-repo setup (optional but recommended)
cd /your/repo
gortex init          # writes .mcp.json, CLAUDE.md, editor configs, guard rules
gortex init --skills # also generates per-community SKILL.md files

# MCP server (standalone mode, auto-detects running daemon)
gortex mcp --index /path/to/repo --watch

# Named project
gortex status
```

After `gortex init`, Claude Code and Cursor start the MCP server automatically via `.mcp.json` — you don't need to run `gortex mcp` manually. The web UI is served separately by `gortex server` (port 4747) and lives at [gortexhq/web](https://github.com/gortexhq/web).

Source: [github.com/zzet/gortex](https://github.com/zzet/gortex). Issues and PRs welcome.

---

## Common questions, Gortex edition

If you landed here looking for GitNexus answers, here's the Gortex equivalent for each.

**Setup / how to use.** `curl -fsSL https://get.gortex.dev | sh` (or `brew install zzet/tap/gortex`), then `gortex install` once per machine, `gortex daemon start --detach` to start the daemon, and `gortex init` once per repo. The MCP config, editor integrations, and guard rules write themselves.

**MCP server / MCP commands.** `gortex mcp --index /path/to/repo --watch`. After `gortex init`, Claude Code and Cursor start it automatically via `.mcp.json` — you don't need to run it manually. The daemon (`gortex daemon start`) keeps the graph live across sessions; `gortex mcp` auto-detects it.

**CLI.** `gortex --help` lists everything. Common ones: `gortex mcp`, `gortex init`, `gortex daemon`, `gortex track`, `gortex status`, `gortex savings`, `gortex eval`, `gortex enrich`.

**Skills.** `gortex init --skills` (or `gortex init` followed by `gortex init --skills` separately) clusters the codebase into functional communities and writes a `SKILL.md` for each. Claude Code picks them up automatically — no extra config.

**OpenCode.** `gortex init` detects OpenCode and writes `.opencode/config.json` automatically. Run it once and the integration is done.

**Antigravity.** `gortex init` adds a knowledge item that tells Antigravity to use Gortex CLI queries instead of grep. Same one-command setup as the other editors.

**"cannot get /".** If you hit a routing error on the GitNexus web UI, the Gortex web UI is a separate app at [gortexhq/web](https://github.com/gortexhq/web). Start the API server with `gortex server --index /path/to/repo --watch` (port 4747), then open the web UI against `http://localhost:4747`.
