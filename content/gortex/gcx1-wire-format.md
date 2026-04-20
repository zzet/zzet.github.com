---
title: "GCX1 — a compact wire format for MCP tool responses"
description: "Median −27.4% tiktoken savings vs JSON, 100% round-trip integrity, across 20 representative MCP tool responses. Shipped, stable, reference decoders in Go and TypeScript."
date: 2026-04-20
lastmod: 2026-04-20
draft: false
slug: "gcx1"
keywords: ["gcx1", "mcp wire format", "mcp token savings", "tiktoken", "mcp tool response", "compact mcp", "gortex wire format", "llm token optimization", "mcp protocol", "tsv mcp"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

Median −27.4% tiktoken savings vs JSON, 100% round-trip integrity, across 20 representative MCP responses. Shipped, stable, reference decoders in Go and TypeScript.

MCP tool responses are mostly tabular: a `search_symbols` call returns N rows of the same shape; `find_usages` returns N edges; `analyze` returns N findings. JSON is a terrible format for tabular data — you pay for every key name on every row, and the LLM pays for the tokens. For a 20-row search result you're repeating `"file_path"`, `"start_line"`, `"kind"` twenty times.

I built GCX1 to fix this for the [Gortex MCP server](https://github.com/zzet/gortex) and then realised the format is tool-agnostic. It's now opt-in on a per-call basis via `format: "gcx"`. Here's what it looks like, why it saves what it saves, and where it doesn't help.

## Concrete example

A `search_symbols` call returning 5 hits.

JSON (720 bytes, 181 tiktoken tokens):

```json
[
  {"id":"internal/mcp/server.go::NewServer","kind":"function","name":"NewServer","file_path":"internal/mcp/server.go","start_line":62},
  {"id":"internal/mcp/server.go::Server.Start","kind":"method","name":"Start","file_path":"internal/mcp/server.go","start_line":118},
  {"id":"internal/mcp/server.go::Server.Shutdown","kind":"method","name":"Shutdown","file_path":"internal/mcp/server.go","start_line":140},
  {"id":"internal/mcp/tools_core.go::registerCoreTools","kind":"method","name":"registerCoreTools","file_path":"internal/mcp/tools_core.go","start_line":307},
  {"id":"internal/mcp/tools_coding.go::registerCodingTools","kind":"method","name":"registerCodingTools","file_path":"internal/mcp/tools_coding.go","start_line":1}
]
```

GCX1 (522 bytes, 125 tokens — −27.5% / −30.9%):

```
GCX1 tool=search_symbols fields=id,kind,name,path,line,sig rows=5 total=5 truncated=false
internal/mcp/server.go::NewServer	function	NewServer	internal/mcp/server.go	62
internal/mcp/server.go::Server.Start	method	Start	internal/mcp/server.go	118
internal/mcp/server.go::Server.Shutdown	method	Shutdown	internal/mcp/server.go	140
internal/mcp/tools_core.go::registerCoreTools	method	registerCoreTools	internal/mcp/tools_core.go	307
internal/mcp/tools_coding.go::registerCodingTools	method	registerCodingTools	internal/mcp/tools_coding.go	1
```

Same information. The header declares the column order once; each row is a single tab-separated line.

## Design

Four things matter.

**1. Tokenizer-aware, not just byte-aware.**

GCX1 is designed around tiktoken `cl100k_base`, the tokenizer Claude and GPT-4-class models use. Tab (`\t`) is a single token. Newline is a single token. `"file_path":"` is 4 tokens. Removing 20 copies of that one repeated key saves 80 tokens, not 20 bytes worth. The benchmark measures tiktoken specifically because that's the number in the user's bill — not raw UTF-8 bytes, which gzip already flattens.

**2. Round-trippable by construction.**

Every GCX1 payload decodes to an equivalent JSON value and re-encodes to byte-identical GCX. Agents that already work against JSON don't break. Decoder implementations: `pkg/wire` (Go) and `@gortex/wire` (TypeScript on npm). 20/20 round-trip in the benchmark — not a sampled claim.

**3. Tab, not comma.**

Symbol signatures contain commas (`(int, string)`) and parentheses. They don't contain tabs. CSV would demand quoting on almost every row; TSV doesn't. The escape alphabet is two bytes: `\t`, `\n`, `\\`. Everything else flows through unescaped.

**4. Versioned and fallback-safe.**

The literal prefix `GCX1` is in every header. A decoder that sees `GCX2` in the future must fall back to JSON by re-issuing the MCP call without `format: "gcx"`. Field layouts for each declared tool are frozen within a major version — additions ship as a new version, not silent schema drift.

## Grammar

```
payload   = section { section } ;
section   = header row-line { row-line | comment } ;
header    = "GCX1" SP "tool=" token { SP key-value } SP "fields=" field-list LF ;
row-line  = value { TAB value } LF | LF ;
value     = { "\\\\" | "\\t" | "\\n" | any-byte-except-TAB-LF-BACKSLASH } ;
```

Multi-section payloads concatenate back-to-back — tools like `get_callers` emit two sections (`.nodes` then `.edges`) so you don't denormalise a graph into row duplication.

Full spec: [docs/wire-format.md](https://github.com/zzet/gortex/blob/main/docs/wire-format.md)

## Benchmark — full scorecard

Reproducible from `bench/wire-format/` with `go run ./bench/wire-format`. Harness captures raw UTF-8 bytes, tiktoken tokens, gzip-compressed bytes, and round-trip integrity across 20 representative tool responses.

| case | tokens JSON | tokens GCX | Δ% |
|------|------------:|-----------:|----:|
| search_symbols (small) | 181 | 125 | −30.9% |
| search_symbols (large) | 1068 | 742 | −30.5% |
| batch_symbols | 335 | 241 | −28.1% |
| find_usages (large) | 569 | 351 | −38.3% |
| analyze_hotspots | 506 | 318 | −37.2% |
| smart_context | 471 | 299 | −36.5% |
| analyze_dead_code | 288 | 198 | −31.2% |
| find_implementations | 224 | 157 | −29.9% |
| get_callers | 750 | 532 | −29.1% |
| get_editing_context | 233 | 171 | −26.6% |
| get_file_summary | 567 | 431 | −24.0% |
| find_usages (small) | 335 | 255 | −23.9% |
| contracts.list | 580 | 463 | −20.2% |
| get_test_targets | 311 | 262 | −15.8% |
| get_repo_outline | 237 | 224 | −5.5% |
| get_symbol_source (small) | 257 | 255 | −0.8% |
| get_symbol_source (large) | 534 | 532 | −0.4% |
| find_cycles | 87 | 90 | +3.4% |
| graph_stats | 162 | 174 | +7.4% |

**Median: −27.4% tokens. Round-trip: 20/20.**

## Where GCX1 doesn't help

Three cases are net-neutral-to-worse, and they're structural, not bugs.

`graph_stats` (+7.4%). A single scalar object with ~6 keys. The GCX1 header is fixed-cost; below ~5 rows the header eats the savings. JSON is fine for this.

`find_cycles` (+3.4%). 3–4 rows of short scalars. Same problem.

`get_symbol_source` (~0%). The payload is dominated by a source-code string field. Neither encoding compresses source text.

The wins come from tabular payloads with repeated shape. That's most MCP tool responses, but not all. The format is opt-in per call for exactly this reason: encoders fall back to JSON for shapes where GCX1 would lose.

## GCX1 vs TOON

[TOON](https://github.com/toon-format/toon) (Token-Oriented Object Notation) solves an adjacent problem and is worth comparing directly.

TOON is a general-purpose compact encoding for LLM input: YAML-style indentation, bracket notation `[N]` for array lengths, and curly-brace field headers `{id,name,value}` with CSV-style rows beneath. It handles nested structures, semi-uniform arrays, and mixed data — and its benchmarks show 33–59% token reduction vs JSON depending on shape.

GCX1 is narrower. It targets MCP tool responses specifically, carries tool-level metadata (`tool=`, `rows=`, `total=`, `truncated=`) in the header, and is flat-only by design. Multi-section payloads (`.nodes` + `.edges`) handle graph structure without nesting.

| | TOON | GCX1 |
|---|---|---|
| Scope | General LLM input | MCP tool responses |
| Nested structures | Yes | No (multi-section instead) |
| Separator | Comma or tab | Tab only |
| Array length declaration | Yes (`[N]`) | Via `rows=N` in header |
| Tool metadata | No | Yes (`tool=`, `total=`, `truncated=`) |
| Round-trippable | Not a design goal | Yes, by construction |
| Versioning + fallback | No | `GCX1` prefix, decoder falls back on `GCX2` |
| Claimed savings | 30–60% (varied data) | 27.4% median (MCP tool shapes) |
| MCP integration | Third-party server | Native `format: "gcx"` per call |

The tab-only choice in GCX1 is the most relevant structural difference. Code signatures — the dominant string type in Gortex responses — contain commas (`func(int, string) error`) but never tabs. TOON supports tab as an alternative separator, but the default is comma with quoting, which adds overhead on exactly the values that appear most in code intelligence payloads. GCX1's escape alphabet is two bytes (`\t`, `\n`, `\\`) and nothing in a symbol signature triggers it.

The other meaningful difference is round-trip. TOON is designed as a format you send to an LLM — the LLM reads it, not a downstream tool. GCX1 payloads need to decode back to typed rows so one tool's output can feed another tool's input without a re-issue. That's an MCP-specific requirement that TOON doesn't have to care about.

If you're building a general agent system that sends varied JSON — user records, event logs, config objects — TOON is worth benchmarking. If you're building an MCP server that returns tabular symbol data, TOON's nesting overhead and comma default are costs you don't need to pay.

## Try it

**MCP server side — any Go server.**

```sh
go get github.com/zzet/gortex/pkg/wire
```

`pkg/wire` is a standalone MIT-licensed Go module, stdlib-only, 250 lines. The rest of Gortex ships under a separate source-available license; the wire package is MIT so you can drop it into any MCP server without license friction.

**Agent side — TypeScript.**

```bash
npm install @gortex/wire
```

```typescript
import { decode } from "@gortex/wire";
const rows = decode(mcpResponseText);
// rows is the same shape you'd get from JSON.parse
```

**Agent side — Go.**

Use the same `pkg/wire` package; `wire.Decode(payload)` returns typed rows.

**Reproduce the benchmark.**

You can run benchmark against of [Gortex](https://github.com/zzet/gortex) optimised JSON responses.

```bash
git clone https://github.com/zzet/gortex
cd gortex
go run ./bench/wire-format
cat bench/wire-format/scorecard.md
```

The fixture set is under `bench/wire-format/cases/`. Each is a YAML file with a canonical JSON payload. Add your own tool's fixtures, run the harness, see what GCX1 does on your shapes.


## Why this exists

I built GCX1 for [Gortex](/gortex/from-gitnexus-to-gortex/), a code-intelligence MCP server that indexes repos into an in-memory knowledge graph and exposes it to Claude Code, Cursor, Windsurf, Copilot, and 11 other AI agents. The server ships back a lot of tabular data — symbols, callers, contracts, dependencies — and the token bill on every user's Claude account was the first thing I wanted to cut. A `compact: true` text mode existed but wasn't round-trippable; you couldn't feed the output back into another tool. Something had to carry the schema without repeating it on every row.

## What's next

GCX1 stays stable. Field layouts for each tool are frozen for the lifetime of GCX1. Adding a column to a tool is a breaking change and ships as GCX2.

GCX1-stream is reserved. Row-at-a-time streaming over SSE / chunked HTTP, same header, same row grammar. Not shipped yet.

GCX2 if needed. If someone wants CBOR or MessagePack-backed rows for binary payloads, that's a future major version. GCX1 is text-only so agents can read raw payloads during debugging — that's a non-negotiable for v1.

If you're building an MCP server: pick your biggest tabular response, measure it through tiktoken, then run it through the same encoder and measure again. If your shape is tabular, you're leaving 25–35% on the table.

---

Spec: [docs/wire-format.md](https://github.com/zzet/gortex/blob/main/docs/wire-format.md)  
Benchmark: [bench/wire-format](https://github.com/zzet/gortex/tree/main/bench/wire-format)  
Go reference: [pkg/wire (MIT)](https://github.com/zzet/gortex/tree/main/pkg/wire)  
TypeScript decoder: [@gortex/wire on npm](https://www.npmjs.com/package/@gortex/wire)
