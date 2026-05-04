# CLI Schema

**The open standard for describing command-line interfaces.**

CLI Schema is a language-agnostic specification for machine-readable CLI descriptions — structured enough for AI agents to reason about, expressive enough to generate documentation, and precise enough to power shell completions and IDE tooling.

Think of it as OpenAPI for CLIs.

---

## Quick start

If your CLI supports the `__schema` meta-command, run:

```sh
mytool __schema
```

Or look for a sidecar file next to the binary:

```sh
cat $(which mytool).cli-schema.json
```

---

## Specification

- [**Specification v1**](spec/v1/README.md) — the authoritative document
- [**Meta-schema**](schema/cli-schema.meta-schema.json) — JSON Schema for validating a CLI Schema document

## Examples

- [`gh.cli-schema.json`](spec/v1/examples/gh.cli-schema.json) — GitHub CLI
- [`git.cli-schema.json`](spec/v1/examples/git.cli-schema.json) — Git

---

## Website

[cli-schema.github.io/cli-schema](https://cli-schema.github.io/cli-schema)

---

## Status

**Draft** — the specification is in active development. Breaking changes may occur before v1.0.0 is declared stable.

Feedback and contributions welcome via [issues](https://github.com/cli-schema/cli-schema/issues).

## License

[CC0 1.0 Universal](LICENSE) — public domain dedication.
