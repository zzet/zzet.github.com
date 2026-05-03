---
title: "Gortex: Week 2 — DI graph, 3D web UI, dependency graph, enrichment, SQL, and 50 MCP tools"
description: "Two weeks of shipping since the last post: DI framework support (NestJS/FastAPI/Spring/Laravel/Rails), react-three-fiber 3D web UI, gortex eval CLI, external package graph, gortex enrich blame+coverage, six new analyzers, deep Go graph, SQL migration analysis, and feature flags."
date: 2026-05-03
lastmod: 2026-05-03
draft: false
slug: "gortex-week-2"
keywords: ["gortex", "mcp", "code intelligence", "dependency graph", "git blame mcp", "code coverage mcp", "sql mcp", "goroutine graph", "gortex enrich", "50 mcp tools", "nestjs mcp", "dependency injection mcp", "gortex eval", "react three fiber graph"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

Two weeks of shipping since the [last post](/gortex/gortex-update-and-comparison/), ~100 commits. The tool count went from 44 to 50 and the graph got significantly denser. Here's what shipped.

## DI framework support

Dependency injection is now a first-class graph concept. Gortex extracts provider/consumer edges from DI decorators and annotations across six ecosystems:

- **NestJS** — `@Injectable`, `@Inject(TOKEN)`, `@Module` with `providers`/`imports`, dynamic modules (`forRoot`, `forRootAsync`), field injection
- **TypeScript** — DI-token recall via `@Inject(TOKEN)` edges, dispatch resolution for decorated class methods
- **FastAPI** — `Depends()` injection sites linked to their dependency callables
- **Spring** — `@Bean`, `@Autowired`, `@Component`, `@Service` provider/consumer edges
- **Angular** — `inject()` call-site extraction
- **Laravel** — middleware registration, service providers (`register`/`boot`)
- **Rails** — `before_action` filter chains as dispatch edges
- **Phoenix** — plug pipeline dispatch edges
- **Symfony** — `#[AsEventListener]` event handler wiring

What this unlocks: `find_implementations` on a DI token returns every class that provides or consumes it. Blast radius on a service interface follows injection edges across module boundaries. Contract detection recognizes DI-injected HTTP clients as consumers of the routes they call.

## Web UI rebuilt on react-three-fiber

The web UI at [gortexhq/web](https://github.com/gortexhq/web) got a full rebuild. The 2D Sigma.js graph is unchanged, but the five 3D views are now on react-three-fiber: **City** (symbol skyline by fan-in), **Strata** (layered by dependency depth), **Galaxies** (community clusters as star systems), **Constellation** (call graph as constellation map), **Graph3D** (force-directed 3D). All five share a common r3f helper layer and consume real `/v1/*` data — no mocks.

The contracts panel got request/response type tracing: provider and consumer drift is shown side-by-side with the inferred type shapes. The processes panel got collapsible call-tree steps and a product/test split so execution flows from tests don't mix with production paths.

## `gortex eval` CLI

A first-class eval harness is now built into the binary:

```bash
gortex eval recall       # fixture-driven retrieval: R@1/5/20, MRR, p50/p95 latency, tokens returned
gortex eval embedders    # compare ONNX variants on size, init time, embed latency, end-to-end quality
gortex eval swebench     # SWE-bench passthrough with Docker environments
gortex eval tokens       # GCX1 wire-format benchmark (same as bench/wire-format/)
```

Published baseline on Gortex's own codebase: **R@1 42.3% · R@5 56.4% · R@20 69.9% · exact R@5 95.2%**. The seed fixture is at `bench/fixtures/retrieval.yaml` — add your own cases and run `gortex eval recall` to see per-case breakdown.

## Search: per-session learning and typo rescue

Search now learns within a session. When an agent picks a result, that choice feeds a combo/frecency score that re-ranks future searches toward symbols the agent has found useful. Opt-in typo rescue (`--typo-rescue`) applies fuzzy matching as a fallback when exact and BM25 searches return nothing.

## Hooks: graph-aware PreToolUse

The Claude Code `PreToolUse` hook now routes `codebase-search`, `rg`, and `grep` probes through Gortex graph tools instead of filesystem traversal. A Bash pattern matcher was added so shell-invoked ripgrep calls are also intercepted. Spawned `Task` subagents get an inline tool-swap table so they don't fall back to grep either.

## GCX1 in standalone repos

The GCX1 reference implementations moved to their own repositories in the `gortexhq` org:

- Go: [github.com/gortexhq/gcx-go](https://github.com/gortexhq/gcx-go)
- TypeScript: [github.com/gortexhq/gcx-ts](https://github.com/gortexhq/gcx-ts) / npm [`@gortex/wire`](https://www.npmjs.com/package/@gortex/wire)

The `pkg/wire` nested module inside Gortex is still MIT-licensed and still works, but the canonical standalone implementation is now in the org repos — easier to depend on without pulling in Gortex itself.

## External packages in the graph

Go imports, npm packages, Python PyPI dependencies, Rust crates, and Maven coordinates are now first-class graph nodes. Gortex parses `go.mod`, `package.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `pyproject.toml`, `requirements.txt`, `Cargo.toml`, and `pom.xml` — resolves each to a dependency node with the exact version from the lockfile, then links it to every symbol that imports it.

What this means in practice: blast radius analysis now surfaces external package version changes as a contributing factor. `get_dependents` on a Go module node returns every symbol that imports it. Contract detection can trace an HTTP route through your Go handler to the underlying third-party client library.

Go imports get a further step: each import is linked to its resolved module node, so `find_usages` on an import path returns every file that uses it — with no filesystem traversal.

## `gortex enrich`

A new offline enrichment command stamps additional metadata onto the graph after indexing.

```bash
gortex enrich blame      # last-author + last-modified-date on every symbol via git blame
gortex enrich coverage   # coverage_pct per symbol from a Go cover profile
gortex enrich all        # run both in one pass, write enriched snapshot to disk
```

The enriched data surfaces in several places. `analyze kind=ownership` uses blame to aggregate review routing — returns which team members own the most code in a blast radius. `analyze kind=coverage_gaps` uses the coverage data to rank undertested symbols by fan-in, which is a much more useful list than raw uncovered lines.

Both enrichments write to the on-disk snapshot so they survive daemon restarts. Useful for CI: index, enrich, start the server, and agents get annotated symbols from the first call.

## Six new `analyze` kinds

The unified `analyze` tool grew six new kinds:

- **`todos`** — surfaces every TODO/FIXME/HACK comment in the graph, grouped by symbol and community. Useful for cleanup triage before a release.
- **`stale_code`** — symbols unchanged for the longest time, weighted by fan-in. Not dead code, but a useful proxy for forgotten complexity.
- **`ownership`** — blame-aggregated ownership per symbol and community. Requires `gortex enrich blame` first.
- **`coverage_gaps`** — symbols not reached from any test file, ranked by fan-in. Requires `gortex enrich coverage` first.
- **`orphan_tables`** — SQL tables that appear in migrations but have no corresponding query-string reference. Catches tables created but never read or written to in Go code.
- **`cgo_users`** — Go files that use cgo, with the C symbols they bind. Useful for security audits and portability checks.

## SQL analysis

The graph now understands SQL. Three layers:

**Migration parsing.** `CREATE TABLE` statements in migration files are extracted as table nodes. The SQL dialect is inferred from Go driver imports (`database/sql` + `lib/pq` → Postgres, `go-sqlite3` → SQLite, etc.) so table IDs are dialect-qualified.

**String-literal query extraction.** SQL query strings in Go source are parsed and linked to the table nodes they reference. A `SELECT * FROM users JOIN orders` in a Go file creates `references` edges from that function to both the `users` and `orders` nodes.

**Orphan detection.** `analyze kind=orphan_tables` crosses the two: tables with a `CREATE TABLE` migration node but zero query-string references. The kind of dead schema that accumulates over years of feature removals.

## Deeper Go graph

Several Go-specific node and edge kinds landed:

**Goroutines.** `go func()` spawn sites are now a distinct edge kind (`EdgeSpawns`) to inline closures. `go someFunc()` creates a `spawns` edge to the target. Channel send (`ch <-`) and receive (`<- ch`) sites are extracted as separate graph events.

**Constants and enums.** Previously these were `KindVariable`. They're now split: `KindConst` for `const` declarations and `KindEnumMember` for iota blocks. This makes dead code analysis more precise — an unused constant is different from an unused variable.

**Type structure.** Type aliases, newtypes (`type Foo Bar`), and embedded composition are now distinguished from each other. A type alias carries an edge to its target; an embedding carries a `member_of` edge. This affects interface satisfaction inference and blast radius accuracy.

**Fields.** Struct fields are first-class graph nodes. `find_usages` on a field ID returns every access site. This was the last major gap in Go graph completeness.

**Method resolution.** Method calls are now resolved by tracking the receiver type from variable declarations, composite literals, and constructor conventions. `s.Handler()` where `s` was declared as `*http.Server` resolves to `http.Server.Handler`, not an ambiguous name match.

**Function shape.** Function signature shapes (parameter and return types) are modeled as graph nodes, which enables queries like "find all functions that return an error and are never checked by their caller."

## Feature flags, config keys, log events

Three new node kinds for cross-cutting concerns:

**Feature flags.** Check sites (`if flags.IsEnabled("my-feature")`, LaunchDarkly, Unleash, homegrown patterns) are extracted as flag nodes with `references` edges from every function that checks them. Deleting a flag: `find_usages` on the flag node gives the full blast radius.

**Config keys.** Viper config-key references (`viper.GetString("db.host")`) land as config nodes. Agents can query which parts of the codebase depend on a specific config key.

**Log emit sites.** `log.Info("payment processed", ...)` calls are extracted as event nodes. Useful for tracing the observability surface of a system and for finding log calls that should be structured but aren't.

TODOs, SPDX license headers, and CODEOWNERS entries also land as graph nodes now. Machine-generated files get a `generated_by` tag so analyzers can skip or surface them explicitly. Testdata files get a distinct `testdata` kind.

## HTTP API

`gortex server` now exposes all MCP tools as a versioned HTTP/JSON API:

```bash
gortex server --index /path/to/repo --watch
# Serves http://localhost:4747/v1/*
```

Endpoints: `/v1/health`, `/v1/tools`, `/v1/tools/{name}` (POST any tool), `/v1/stats`, `/v1/graph`, `/v1/events` (SSE for live graph changes). Non-localhost binds require `--auth-token`. CORS configurable. Unix socket supported (`--bind unix:///path/to.sock`).

The web UI lives at [gortexhq/web](https://github.com/gortexhq/web) and talks to this API — Sigma.js 2D graph, five Three.js 3D views, symbol search, community explorer, contract checker, and analysis tabs.

## New MCP tools

Four new tools crossed the 50 total:

**`winnow_symbols`** — structured multi-axis retrieval. Filter by kind, language, community, path prefix, minimum fan-in, minimum fan-out, minimum churn, and text match simultaneously, with per-axis score contributions. More precise than `search_symbols` when you know the shape of what you want.

**`plan_turn`** — opening-move router. Given a task description, returns a ranked list of next tool calls with pre-filled arguments, ~200 tokens. Useful as the first call in an agent session when the task is ambiguous.

**`get_untested_symbols`** — the inverse of `get_test_targets`. Returns functions and methods not reached from any test file, ranked by fan-in. The ranking matters: an untested function called from 40 places is more important than one called from 2.

**`edit_file` / `write_file`** — read-free file writes. `edit_file` does exact string replacement on any file (source, markdown, config, spec) without a prior Read. `write_file` creates or overwrites. Both re-index on write. Eliminates the Read-before-Edit round-trip on files outside the symbol graph.

## Multi-repo workspace management

`gortex init` now binds per-repo config to `.gortex/` in the repo root. `gortex workspace set/set-all` stamps workspace and project slugs across tracked repos in bulk — useful when migrating an existing multi-repo setup to named workspaces.

The daemon's multi-server roster (`gortex daemon server add/remove`) lets a single daemon route across local and remote Gortex instances. A local Unix socket for repos on this machine, a remote HTTPS server for a shared cloud index — the daemon picks the right target per query.

MCP indexing now emits `notifications/progress` during long operations (walking → parsing → resolving → semantic enrichment → search index → contracts → done), so hosts with progress bar support show actual stage information on large repos.

## Rust wasm-bindgen and Go import bridging

Rust files using `#[wasm_bindgen]` are detected and routed through a shared interop handler that models the JS↔Wasm boundary as a contract. Go imports are now bridged to `dep::<module>` contract nodes, which means `check_contracts` can detect a Go file importing a package that isn't declared as a dependency in `go.mod`.

---

Source: [github.com/zzet/gortex](https://github.com/zzet/gortex)

```bash
brew install zzet/tap/gortex   # macOS
gortex install                  # once per machine
gortex init                     # once per repo (optional)
```
