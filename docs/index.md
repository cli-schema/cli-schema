---
layout: home
title: CLI Schema
---

<div class="hero">
  <h1>CLI Schema</h1>
  <p class="tagline">The open standard for describing command-line interfaces.</p>
  <p class="sub">Machine-readable. Language-agnostic. Built for agents, IDEs, shells, and docs.</p>
  <div class="cta-group">
    <a href="/cli-schema/spec.html" class="btn btn-primary">Read the spec</a>
    <a href="https://github.com/cli-schema/cli-schema" class="btn btn-secondary">GitHub</a>
  </div>
</div>

---

## The problem

CLIs are the most ubiquitous programmable interface in software — and the most opaque to tooling.

- **AI agents** guess flags and parameters. They hallucinate options that don't exist.
- **IDE extensions** hand-author completion plugins per tool, per shell, per version.
- **Documentation** is written by hand and drifts from the code.
- **Shell completions** require bespoke scripts that no one keeps up to date.

There is no standard way for a CLI to say *what it can do* without running it.

---

## The solution

A single JSON document that any CLI can emit or ship alongside its binary.

```sh
# Invoke the meta-command
mytool __schema

# Or read the sidecar file without spawning a process
cat $(which mytool).cli-schema.json
```

One format. All consumers.

---

## What it describes

<div class="feature-grid">
  <div class="feature">
    <h3>Commands &amp; namespaces</h3>
    <p>Full command tree with names, aliases, summaries, usage examples, and nested namespace groups.</p>
  </div>
  <div class="feature">
    <h3>Parameters</h3>
    <p>Flags and positionals with types, defaults, validation constraints, aliases, and whether they're required or repeatable.</p>
  </div>
  <div class="feature">
    <h3>Intent model</h3>
    <p>Is the command destructive? Idempotent? Does it affect a file, a directory tree, or a global resource? Does it block on user confirmation?</p>
  </div>
  <div class="feature">
    <h3>Dry-run &amp; confirmation</h3>
    <p>First-class annotations mark which flags skip interactive prompts (<code>--yes</code>, <code>--force</code>) and which enable safe preview mode (<code>--dry-run</code>).</p>
  </div>
  <div class="feature">
    <h3>Auth &amp; environment</h3>
    <p>Which commands require authentication, which commands handle login, and what environment variables or config files the tool depends on.</p>
  </div>
  <div class="feature">
    <h3>Output formats</h3>
    <p>Which output formats are supported (JSON, table, CSV) and which flag selects them — so consumers can request structured output reliably.</p>
  </div>
  <div class="feature">
    <h3>Deprecation</h3>
    <p>Commands and flags carry structured deprecation metadata: what to use instead, when it was deprecated, when it will be removed.</p>
  </div>
  <div class="feature">
    <h3>Vendor extensions</h3>
    <p>Any field prefixed with <code>x-</code> is a free-form extension. Consumers ignore unknown extensions; implementations can annotate anything.</p>
  </div>
</div>

---

## Who it's for

<div class="consumer-grid">
  <div class="consumer">
    <h3>🤖 AI agents</h3>
    <p>Agents read the schema before invoking a CLI. They understand what's safe to call autonomously, which flags are destructive, whether they need to authenticate first, and how to run non-interactively.</p>
  </div>
  <div class="consumer">
    <h3>🛠 IDE tooling</h3>
    <p>Editors offer autocomplete, hover documentation, and inline validation for CLI arguments — without a per-tool plugin, without running the process.</p>
  </div>
  <div class="consumer">
    <h3>💻 Shell completions</h3>
    <p>A single schema generator can produce bash, zsh, fish, and PowerShell completions for any conforming CLI. No bespoke completion scripts.</p>
  </div>
  <div class="consumer">
    <h3>📖 Documentation</h3>
    <p>Reference documentation, man pages, and web docs generated directly from the schema — always accurate, never hand-maintained.</p>
  </div>
</div>

---

## Quick example

```json
{
  "schemaVersion": 1,
  "name": "gh",
  "version": "2.45.0",
  "description": "GitHub CLI",
  "requiresAuth": true,
  "authCommands": ["auth login"],
  "namespaces": [{
    "segment": "repo",
    "commands": [{
      "name": "delete",
      "summary": "Delete a GitHub repository",
      "tags": ["admin", "dangerous"],
      "intent": {
        "destructive": true,
        "idempotent": false,
        "scope": "global",
        "requiresConfirmation": true,
        "requiresAuth": true
      },
      "parameters": [{
        "role": "confirmationSkip",
        "name": "yes",
        "type": "boolean",
        "required": false,
        "summary": "Skip confirmation prompt"
      }]
    }]
  }]
}
```

An agent reading this knows: the command is destructive, requires auth, will block on stdin unless `--yes` is passed, and affects a global resource (the repository on GitHub). It can make an informed decision about whether to proceed and how.

---

## Status

**Draft** — the specification is in active development. Feedback and contributions are welcome via [GitHub Issues](https://github.com/cli-schema/cli-schema/issues).
