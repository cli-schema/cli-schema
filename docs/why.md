---
layout: page
title: Why CLI Schema?
permalink: /why/
---

## The state of CLI tooling

CLIs are the backbone of developer tooling. Git, Docker, kubectl, the AWS CLI, GitHub CLI — nearly every infrastructure product exposes its primary programmable interface as a CLI. And yet CLIs remain opaque to the tools developers use to work with them.

### Help text is for humans, not machines

Every CLI ships a `--help` flag. The output is designed for a person to read in a terminal. It's unstructured text. Parsing it is fragile, format-dependent, and yields no semantic information — you can extract flag names, but not their types, constraints, default values, or what they actually do to the system.

### Shell completions are bespoke and drift

Shell completions for popular CLIs are usually hand-authored, checked into the CLI's repo (or a separate completions repo), and maintained manually. They're per-shell. They lag behind the actual CLI. They frequently omit subcommands added in recent releases. There is no shared format, no way to generate them mechanically from the source of truth.

### AI agents are flying blind

AI agents that orchestrate CLIs — running `git push`, creating GitHub repos, deploying to Kubernetes — have no reliable way to discover what options a command accepts, whether a command is destructive, whether it will block on stdin waiting for confirmation, or what environment it needs to be set up. Agents hallucinate flags. They accidentally delete things. They get stuck waiting for interactive prompts.

### Documentation drifts

Reference docs for CLIs are often generated from code once and then maintained separately. Over time they diverge. Users read docs describing flags that no longer exist, or miss flags added in recent versions.

---

## What OpenAPI did for HTTP APIs

OpenAPI (formerly Swagger) created a standard machine-readable format for HTTP APIs. The ecosystem that grew around it is enormous: client generators in every language, mock servers, documentation sites, API explorers, contract testing tools, linters, SDK generators.

Before OpenAPI, every HTTP API had its own documentation format (or no machine-readable documentation at all). After OpenAPI, tooling authors had one target to build against.

CLIs need the same thing.

---

## Why now?

Three forces are converging:

**AI agents orchestrating CLIs.** As LLMs become capable of autonomously running terminal commands, the question "is this command safe to run?" becomes critical. A schema that encodes intent (destructive, idempotent, scope) and agent-safe flags (confirmation-skip, dry-run) is exactly what an agent needs to reason about before acting.

**Language server infrastructure.** The success of LSP (Language Server Protocol) has made it normal to expect that tools provide structured metadata about their surface area. IDEs already expect machine-readable completion sources. A standard CLI schema is the missing piece.

**Source generators.** Modern CLI frameworks (including [Argh](https://github.com/nullean/argh), which pioneered this format) emit schema at build time from code — no manual maintenance. The schema is always accurate because it's derived from the same source as the CLI itself.

---

## Design principles

**Language-agnostic.** Nothing in the format assumes a specific runtime, language, or framework. A CLI written in Go, Rust, C#, Python, or shell can all emit a conforming document.

**Consumer-agnostic.** One format serves AI agents, IDE tools, shell completions, and documentation generators equally. We don't optimize for one consumer at the cost of others.

**AOT-safe.** Schema documents can be read without spawning the process. The sidecar file mechanism means a sandboxed agent can read the schema of a tool it's not allowed to execute.

**Minimal required surface.** Only three fields are required at the root (`schemaVersion`, `name`, `version`). Everything else is optional enrichment. A minimal schema is better than no schema.

**Extensible.** The `x-` vendor extension convention (from OpenAPI) lets implementations annotate anything without polluting the core spec.
