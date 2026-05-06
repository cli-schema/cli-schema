---
navigation_title: Why CLI Schema?
---

# Why CLI Schema?

## The problem

Consider an AI agent tasked with cleaning up a repository. It needs to delete a GitHub repo. It calls `gh repo delete owner/repo`. No flag was passed to skip confirmation — the command blocks on stdin, the agent times out, and nothing happens. Or worse: it guesses `--force`, which is not a real flag, and errors out. Or it finds `--yes` by parsing help text, passes it, and the repository is gone before a human had a chance to review.

Three distinct failures. The same root cause: **the agent had no structured way to know what the command does before it runs it.**

This is not an AI-specific problem. It affects every consumer of CLIs:

- **Help text is written for humans.** It is freeform prose, inconsistently formatted, and requires a running process to read. Tools cannot reliably parse it.
- **Bespoke completions don't compose.** Shell completion scripts are hand-authored per-shell, per-tool. They live outside the binary, drift from the implementation, and are not useful to non-shell consumers.
- **Documentation drifts.** Generated docs and README examples go stale. There is no canonical machine-readable source of truth.

## The OpenAPI analogy

Before OpenAPI, HTTP APIs had the same problem. Every client had to read documentation, guess field names, and discover error codes at runtime. OpenAPI changed that: one document describes the entire API surface. Clients, mocks, validation libraries, and documentation generators are all built from the same source.

CLI Schema does the same thing for command-line tools. One document describes the entire interface:

| HTTP / OpenAPI | CLI / CLI Schema |
|---|---|
| Endpoints | Commands and namespaces |
| Query params and request bodies | Flags and positional arguments |
| Response schemas | Output formats |
| Auth schemes | Auth requirements and auth commands |
| Idempotency annotations | Intent object |

When a CLI ships a schema document, every CLI Schema-aware consumer understands it immediately. No parsing. No guessing.

## Why now

Three trends make this the right moment for a CLI interface standard:

**AI agents are running CLIs autonomously.** They need to know whether a command is destructive, whether it requires confirmation, and how to request structured output — before running it. Help text is not designed for this.

**LSP infrastructure is everywhere.** IDE tooling, language servers, and shell completion frameworks are already built to consume structured interface definitions. CLI Schema gives them something to consume.

**Source generators are common.** Teams increasingly generate CLIs from interface definitions, making it natural to emit a schema document as part of the same pipeline.

## Design principles

**Language-agnostic.** Nothing about the format assumes a specific runtime, framework, or language. Any tool in any language can emit a valid schema.

**Consumer-agnostic.** One document serves AI agents, IDEs, shell completions, and documentation generators equally. No consumer gets a privileged format.

**AOT-safe.** Schema documents can be shipped as sidecar files and read without spawning the process. This matters in sandboxed environments and for tools that take a long time to start.

**Minimal required surface.** Only three fields are required at the root: `schemaVersion`, `name`, and `version`. Everything else is optional enrichment. An implementation can start minimal and add detail over time.

**Extensible.** Vendor extensions are first-class via the `x-` prefix. Any implementation can add proprietary metadata without breaking conforming consumers.
