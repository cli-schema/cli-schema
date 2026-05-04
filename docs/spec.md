---
layout: page
title: Specification
permalink: /spec/
---

The current specification is **Version 1 (Draft)**.

**[Read Specification v1 →](https://github.com/cli-schema/cli-schema/blob/main/spec/v1/README.md)**

---

## Quick reference

### Discovery mechanisms

| Mechanism | Command / Path |
|---|---|
| Meta-command | `mytool __schema` |
| Sidecar file | `<binary>.cli-schema.json` adjacent to the binary |

### Root object (required fields)

| Field | Type |
|---|---|
| `schemaVersion` | integer — always `1` |
| `name` | string — binary name |
| `version` | string — program version |

### Parameter roles

| Role | Meaning |
|---|---|
| `flag` | Named option (e.g. `--output json`) |
| `positional` | Positional argument |
| `confirmationSkip` | Suppresses interactive confirmation (`--yes`, `--force`) |
| `dryRun` | Enables safe preview mode (`--dry-run`, `--whatif`) |

### Intent fields

| Field | Type | Meaning |
|---|---|---|
| `destructive` | boolean | Deletes or irreversibly modifies |
| `idempotent` | boolean | Safe to retry |
| `scope` | `"file"` \| `"directory"` \| `"global"` | Blast radius |
| `requiresConfirmation` | boolean | Will block on stdin without a `confirmationSkip` flag |
| `requiresAuth` | boolean | Requires authenticated session |

### Validation constraint kinds

`range` · `timeSpanRange` · `length` · `regex` · `allowed` · `denied` · `email` · `url` · `uriScheme` · `fileExtensions` · `existing` · `nonExisting` · `rejectSymbolicLinks`

---

## Examples

- [**gh.cli-schema.json**](https://github.com/cli-schema/cli-schema/blob/main/spec/v1/examples/gh.cli-schema.json) — GitHub CLI
- [**git.cli-schema.json**](https://github.com/cli-schema/cli-schema/blob/main/spec/v1/examples/git.cli-schema.json) — Git

---

## Meta-schema

Validate your schema document against the [JSON Schema meta-schema](https://github.com/cli-schema/cli-schema/blob/main/schema/cli-schema.meta-schema.json).

```sh
# Using ajv-cli
npx ajv validate \
  -s https://cli-schema.github.io/cli-schema/schema/cli-schema.meta-schema.json \
  -d mytool.cli-schema.json
```

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0.0-draft | 2025-05 | Initial draft |
