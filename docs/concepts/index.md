---
navigation_title: Key concepts
---

# Key concepts

A CLI Schema document is a JSON object describing a CLI program's entire interface. This page explains the main building blocks and how they relate.

## Commands and namespaces

Most CLIs expose a tree of subcommands. CLI Schema represents this with two objects:

- A **Command** is a leaf node — something the user actually runs (`git commit`, `gh repo delete`).
- A **Namespace** is a grouping prefix — a word that introduces a set of commands (`repo` in `gh repo ...`).

Namespaces can be nested arbitrarily. The root of the document holds top-level `commands` and `namespaces`.

```
gh
├── auth (namespace)
│   ├── login (command)
│   ├── logout (command)
│   └── status (command)
└── repo (namespace)
    ├── clone (command)
    ├── create (command)
    └── delete (command)
```

## Parameters

Every command has parameters. A parameter has one of four roles:

| Role | What it represents |
|---|---|
| `flag` | A named option: `--output json`, `-v` |
| `positional` | An unnamed argument by position: `gh repo delete <repository>` |
| `confirmationSkip` | A flag that suppresses interactive prompts: `--yes`, `--force` |
| `dryRun` | A flag that makes the command safe to run without side effects: `--dry-run`, `--whatif` |

The `confirmationSkip` and `dryRun` roles are purpose-labeled: a consumer that runs non-interactively knows to look for a `confirmationSkip` parameter and pass it automatically on destructive commands. A consumer that wants to preview what a command would do looks for a `dryRun` parameter.

Parameters also carry type information, validation constraints, aliases, and deprecation status.

## Intent

The **Intent Object** lets a command declare its side-effect profile:

| Field | What it tells a consumer |
|---|---|
| `destructive` | This command deletes, overwrites, or irreversibly modifies something |
| `idempotent` | Running it multiple times is safe — same result as running it once |
| `scope` | Blast radius: `file`, `directory`, or `global` (cloud resource, database, etc.) |
| `requiresConfirmation` | Without a `confirmationSkip` flag, this command will block on stdin |
| `requiresAuth` | This specific command needs an authenticated session |

All intent fields are optional. An omitted field means the implementation has not declared a value. Consumers should treat unknown intent conservatively.

## Discovery

Consumers find a CLI's schema through two mechanisms:

**Meta-command:** The program responds to `<binary> __schema` by printing a valid schema document to stdout and exiting with code 0.

```sh
gh __schema       # prints the full gh CLI schema as JSON
git __schema      # prints the full git CLI schema as JSON
```

**Sidecar file:** A file named `<binary>.cli-schema.json` is shipped next to the binary. Consumers can read it without spawning the process — important in sandboxed or offline environments.

```
/usr/local/bin/gh
/usr/local/bin/gh.cli-schema.json
```

Implementations should support at least one mechanism. Supporting both is recommended.

## Environment

The **Environment Object** declares what the program reads from the environment: variables like `GITHUB_TOKEN` or `GIT_DIR`, and config files like `~/.config/gh/config.yml`. Consumers use this to understand what context they need to set up before running commands.

## Output

The **Output Object** on a command declares what machine-readable formats it supports (e.g. `["json", "table"]`) and which flag selects the format (`--json`, `--output`). A consumer that needs structured data can use this to select the right flag value without parsing help text.

## Deprecation

Commands and parameters can be marked deprecated with a simple boolean or a structured object that includes a message, the version it was deprecated in, and the version it will be removed in. Consumers should surface this information as warnings.

## Vendor extensions

Any field starting with `x-` is a vendor extension. Extensions can appear on any object. Standard consumers ignore them; custom consumers can use them for proprietary metadata.
