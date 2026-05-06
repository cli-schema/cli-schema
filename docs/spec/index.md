---
navigation_title: Specification
---

# Specification

CLI Schema is versioned. This page tracks the current specification status and links to the normative text.

## Current version

**v1.0.0-draft** — initial draft, published May 2025.

The specification is in active development. The core format is stable enough to build against, but field names and semantics may change before a stable v1.0.0 release. Feedback and proposals are welcome via [GitHub Issues](https://github.com/cli-schema/cli-schema/issues).

## How to read the spec

The specification uses [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) keywords:

- **MUST** / **MUST NOT** — required for conformance
- **SHOULD** / **SHOULD NOT** — recommended; deviation must be justified
- **MAY** — optional

Three terms have specific meanings throughout:

- **Schema document** — a JSON object conforming to this specification
- **Implementation** — a CLI program that emits or ships a schema document
- **Consumer** — a tool that reads and acts on a schema document

## Validate a schema document

A JSON meta-schema is provided for mechanical validation. Using `ajv-cli`:

```sh
npx ajv-cli validate \
  -s schema/cli-schema.meta-schema.json \
  -d my-tool.cli-schema.json
```

The meta-schema is published in the repository at `schema/cli-schema.meta-schema.json`.

## Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0.0-draft | May 2025 | Initial draft |

## Full specification

- [CLI Schema v1](v1.md)
