# CLI Schema Specification — Version 1

**Status:** Draft  
**Repository:** https://github.com/cli-schema/cli-schema

---

## Abstract

CLI Schema defines a JSON format for describing command-line interface programs in a language-agnostic, machine-readable way. A conforming CLI Schema document captures a program's commands, parameters, intent, authentication requirements, environment dependencies, and output capabilities in enough detail to support AI agent orchestration, IDE tooling, shell completion generators, and documentation systems — without running the program.

---

## 1. Introduction

CLIs are the most common programmable interface in software, yet they remain opaque to tooling. Help text is written for humans. Completion scripts are hand-authored per-shell. AI agents guess flags. Documentation drifts from the code.

CLI Schema addresses this by defining a single structured document format that implementations emit alongside or from their binary. Consumers parse the document once and gain full knowledge of the program's surface area.

### Design goals

- **Language-agnostic**: nothing about the format assumes a specific runtime or language
- **Consumer-agnostic**: one document serves AI agents, IDEs, shell completions, and docs generators equally
- **AOT-safe**: schema documents can be read without spawning the process (via sidecar file)
- **Extensible**: vendor extensions are first-class via the `x-` prefix convention
- **Minimal required surface**: only a small subset of fields is required; the rest is optional enrichment

---

## 2. Conventions

The keywords **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are used as defined in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

**Schema document**: a JSON object conforming to this specification.  
**Implementation**: a CLI program that emits or ships a schema document.  
**Consumer**: a tool that reads and acts on a schema document.

---

## 3. Document Format

Schema documents MUST be encoded as UTF-8 JSON. Documents SHOULD be formatted with two-space indentation for human readability, though compact (minified) JSON is valid.

The top-level value MUST be a JSON object (not null, array, or scalar).

---

## 4. Discovery

Consumers discover schema documents through two mechanisms. Implementations SHOULD support at least one. Supporting both is RECOMMENDED.

### 4.1 Meta-command (`__schema`)

An implementation that supports this mechanism responds to a single argument `__schema` by printing a valid schema document to stdout and exiting with code 0.

```sh
mytool __schema
```

The `__schema` argument SHOULD appear in the schema document's `reservedMetaCommands` array so consumers know it is available.

Implementations SHOULD NOT treat `__schema` as a subcommand visible in help output.

### 4.2 Sidecar file

An implementation that supports this mechanism ships a file named `<binary>.cli-schema.json` adjacent to the binary executable. For example:

```
/usr/local/bin/gh
/usr/local/bin/gh.cli-schema.json
```

This mechanism allows consumers to read the schema without spawning a process, which is important for sandboxed or untrusted environments.

### 4.3 Package metadata (informative)

Package managers MAY embed a reference to or copy of the schema in their own metadata formats (npm `package.json`, NuGet `.nuspec`, Homebrew formula, etc.). This is outside the scope of this specification but is encouraged as an additional discovery layer.

---

## 5. Root Object

The root of a schema document is a **Root Object**.

### Required fields

| Field | Type | Description |
|---|---|---|
| `schemaVersion` | integer | Always `1` for this version of the specification |
| `name` | string | The binary name of the program (e.g. `"gh"`, `"git"`) |
| `version` | string | The program version (e.g. `"2.45.0"`) |

### Optional fields

| Field | Type | Description |
|---|---|---|
| `description` | string | Short prose description of the program |
| `tags` | string[] | Free-form tags for categorization (e.g. `["vcs", "github"]`) |
| `requiresAuth` | boolean | Whether the program requires authentication to use at all |
| `authCommands` | string[] | Paths of commands that handle login/logout (e.g. `["auth login", "auth logout"]`) |
| `environment` | [Environment Object](#9-environment-object) | Required/optional env vars and config files |
| `reservedMetaCommands` | string[] | Arguments the program reserves for tooling (e.g. `["__schema", "__complete"]`) |
| `globalOptions` | [Parameter Object](#6-parameter-object)[] | Flags available on every command |
| `rootDefault` | [Default Handler Object](#10-default-handler-object) | Handler invoked when the program is called with no subcommand |
| `commands` | [Command Object](#7-command-object)[] | Top-level commands |
| `namespaces` | [Namespace Object](#8-namespace-object)[] | Named groupings of commands (subcommand namespaces) |
| `shortcuts` | [Shortcut Object](#51-shortcut-object)[] | Root-level aliases that transparently resolve to a deeper namespace or command path |

### 5.1 Shortcut Object

A Shortcut Object declares a root-level word that the implementation
transparently rewrites to a full path before routing.

| Field | Type | Description |
|---|---|---|
| `from` | string | The root-level word the user types (e.g. `"es"`) |
| `to` | string[] | The full path it resolves to — may end at a namespace or a command (e.g. `["stack", "es"]` or `["stack", "es", "search"]`) |

Consumers SHOULD treat a shortcut as fully equivalent to its `to` path:
documentation, shell completion, and agent orchestration MAY expose both
forms or either form, but MUST NOT assign different semantics to them.
A shortcut is purely a convenience alias; the canonical path is `to`.

Implementations SHOULD validate at load time that each `to` path resolves
to an existing namespace or command in the document. Consumers that perform
their own validation SHOULD emit a warning (not an error) for unresolvable
shortcuts, since partial documents and forward-declared schemas are valid
use cases.

### Example

```json
{
  "schemaVersion": 1,
  "name": "gh",
  "version": "2.45.0",
  "description": "GitHub CLI — bring GitHub to your terminal",
  "tags": ["github", "vcs", "devtools"],
  "requiresAuth": true,
  "authCommands": ["auth login", "auth logout", "auth status"],
  "reservedMetaCommands": ["__schema"],
  "globalOptions": [],
  "commands": [],
  "namespaces": []
}
```

---

## 6. Parameter Object

A Parameter Object describes a single flag or positional argument.

### Required fields

| Field | Type | Description |
|---|---|---|
| `role` | string | `"flag"`, `"positional"`, `"confirmationSkip"`, or `"dryRun"` — see [6.1 Roles](#61-roles) |
| `name` | string | The parameter name. For flags: the long name without leading `--` (e.g. `"output"`). For positionals: a descriptive label (e.g. `"repository"`) |
| `type` | string | JSON Schema primitive: `"string"`, `"integer"`, `"number"`, `"boolean"`, `"array"`, or `"enum"` |
| `required` | boolean | Whether the parameter must be provided |

### Optional fields

| Field | Type | Default | Description |
|---|---|---|---|
| `shortName` | string | — | Single character short option without leading `-` (e.g. `"o"`) |
| `summary` | string | — | One-line description for help text |
| `defaultValue` | string | — | String representation of the default value |
| `repeatable` | boolean | `false` | Whether the flag may be specified multiple times (collection without separator) |
| `separator` | string | — | Separator for multi-value flags passed as a single value (e.g. `","`) |
| `aliases` | string[] | — | Alternative long names for the flag |
| `enumValues` | string[] | — | Allowed values when `type` is `"enum"` |
| `elementType` | string | — | Type of array elements when `type` is `"array"` |
| `hidden` | boolean | `false` | Whether the parameter is hidden from help output |
| `variadic` | boolean | `false` | Whether the positional captures all remaining arguments as a collection. Only meaningful when `role` is `"positional"`. |
| `deprecated` | boolean \| [Deprecation Object](#111-deprecation-object) | — | Marks the parameter as deprecated |
| `validations` | [Constraint Object](#62-constraints)[] | — | Validation rules applied to the parameter's value |

### 6.1 Roles

| Role | Meaning |
|---|---|
| `"flag"` | A named option (e.g. `--output json`) |
| `"positional"` | A positional argument (e.g. `gh repo delete <repository>`) |
| `"confirmationSkip"` | A flag that suppresses interactive confirmation prompts (e.g. `--yes`, `--force`). Consumers that run non-interactively SHOULD automatically pass this flag on destructive commands. |
| `"dryRun"` | A flag that switches the command to preview/dry-run mode, making it safe to call without side effects (e.g. `--dry-run`, `--whatif`). Consumers exploring what a command would do SHOULD pass this flag. |

### 6.2 Constraints

Constraints apply validation rules to a parameter's value. Each constraint is an object with a required `kind` field and optional additional fields.

| Kind | Additional fields | Description |
|---|---|---|
| `"range"` | `min`, `max` | Numeric range (inclusive). Omit either bound for open ranges |
| `"timeSpanRange"` | `min`, `max` | Duration range in ISO 8601 format (e.g. `"PT5M"`) |
| `"length"` | `min`, `max` | String length in characters |
| `"regex"` | `pattern` | ECMAScript regex pattern the value must match |
| `"allowed"` | `values` | Whitelist of accepted string values |
| `"denied"` | `values` | Blacklist of rejected string values |
| `"email"` | — | Value must be a valid email address |
| `"url"` | — | Value must be a valid URL |
| `"uriScheme"` | `values` | Value must have one of the listed URI schemes (e.g. `["http", "https"]`) |
| `"fileExtensions"` | `values` | Value must have one of the listed file extensions (e.g. `[".json", ".yaml"]`) |
| `"count"` | `min`, `max` | Collection item count (inclusive). Omit either bound for open ranges. Used with variadic positionals. |
| `"existing"` | — | Path must exist on the filesystem |
| `"nonExisting"` | — | Path must not exist on the filesystem |
| `"rejectSymbolicLinks"` | — | Path must not be a symbolic link |

---

## 7. Command Object

A Command Object describes a single executable command.

### Required fields

| Field | Type | Description |
|---|---|---|
| `name` | string | The command name as typed by the user |

### Optional fields

| Field | Type | Default | Description |
|---|---|---|---|
| `path` | string[] | — | Namespace path prefix (e.g. `["repo"]` for `gh repo delete`) |
| `summary` | string | — | One-line description |
| `notes` | string | — | Extended prose description (Markdown) |
| `usage` | string | — | Custom usage synopsis (overrides generated synopsis) |
| `examples` | string[] | — | Example invocations |
| `parameters` | [Parameter Object](#6-parameter-object)[] | — | Command-specific flags and positionals |
| `aliases` | string[] | — | Alternative command names |
| `hidden` | boolean | `false` | Whether the command is hidden from help output |
| `tags` | string[] | — | Free-form tags (e.g. `["admin", "dangerous"]`) |
| `deprecated` | boolean \| [Deprecation Object](#111-deprecation-object) | — | Marks the command as deprecated |
| `intent` | [Intent Object](#11-intent-object) | — | Declares the command's side-effect profile for agent reasoning |
| `output` | [Output Object](#12-output-object) | — | Declares supported output formats |
| `streaming` | boolean | `false` | Whether the command streams output continuously |
| `longRunning` | boolean | `false` | Whether the command is expected to run for a long time (minutes or more) |

---

## 8. Namespace Object

A Namespace Object represents a named grouping of commands (subcommand namespace). Namespaces may be nested arbitrarily.

### Required fields

| Field | Type | Description |
|---|---|---|
| `segment` | string | The namespace word as typed by the user (e.g. `"repo"` in `gh repo ...`) |

### Optional fields

| Field | Type | Description |
|---|---|---|
| `summary` | string | One-line description of the namespace |
| `notes` | string | Extended prose description |
| `options` | [Parameter Object](#6-parameter-object)[] | Flags available on all commands within this namespace |
| `defaultCommand` | [Default Handler Object](#10-default-handler-object) | Handler invoked when the namespace is called with no subcommand |
| `commands` | [Command Object](#7-command-object)[] | Commands within this namespace |
| `namespaces` | Namespace Object[] | Nested namespaces |

---

## 9. Environment Object

Declares external context the program depends on.

### Fields

| Field | Type | Description |
|---|---|---|
| `variables` | [Env Var Object](#91-env-var-object)[] | Environment variables |
| `configFiles` | [Config File Object](#92-config-file-object)[] | Configuration files |

### 9.1 Env Var Object

| Field | Type | Description |
|---|---|---|
| `name` | string | Variable name (e.g. `"GITHUB_TOKEN"`) |
| `required` | boolean | Whether the program requires this variable to function |
| `description` | string | What the variable controls |
| `defaultValue` | string | Default value if not set |

### 9.2 Config File Object

| Field | Type | Description |
|---|---|---|
| `path` | string | File path. `~` is expanded to the user's home directory |
| `description` | string | What the file controls |
| `required` | boolean | Whether the program requires this file to exist |

---

## 10. Default Handler Object

Describes a handler invoked when a program or namespace is called with no subcommand.

| Field | Type | Description |
|---|---|---|
| `kind` | string | `"root"` or `"namespace"` |
| `summary` | string | One-line description |
| `notes` | string | Extended description |
| `usage` | string | Custom usage synopsis |
| `examples` | string[] | Example invocations |
| `parameters` | [Parameter Object](#6-parameter-object)[] | Parameters for this handler |
| `hidden` | boolean | Whether hidden from help |

---

## 11. Intent Object

The Intent Object allows a command to declare its side-effect profile. Consumers — especially AI agents — use this information to reason about risk before executing a command.

| Field | Type | Description |
|---|---|---|
| `destructive` | boolean | `true` if the command deletes, overwrites, or irreversibly modifies data or resources |
| `idempotent` | boolean | `true` if calling the command multiple times produces the same result as calling it once |
| `scope` | string | Blast radius: `"file"` (one file), `"directory"` (a directory tree), or `"global"` (cloud resource, database, registry, or other shared state) |
| `requiresConfirmation` | boolean | `true` if the command will block on an interactive stdin prompt when run without a `confirmationSkip`-role parameter |
| `requiresAuth` | boolean | `true` if this specific command requires an authenticated session |

All fields are optional. An omitted field means the implementation has not declared a value; consumers SHOULD treat omitted fields conservatively (assume unknown risk).

### Example

```json
{
  "intent": {
    "destructive": true,
    "idempotent": false,
    "scope": "global",
    "requiresConfirmation": true,
    "requiresAuth": true
  }
}
```

### 11.1 Deprecation Object

A Deprecation Object provides structured deprecation metadata.

| Field | Type | Description |
|---|---|---|
| `message` | string | Human-readable explanation (what to use instead) |
| `since` | string | Version when the command/parameter was deprecated |
| `removedIn` | string | Version when the command/parameter will be or was removed |

When the `deprecated` field is simply `true`, it signals deprecation without version detail. Consumers SHOULD surface a warning to users for any deprecated command or parameter.

---

## 12. Output Object

Declares the machine-readable output formats a command supports.

| Field | Type | Description |
|---|---|---|
| `formats` | string[] | Supported output format names (e.g. `["json", "table", "csv"]`) |
| `formatFlag` | string | The flag name used to select the format (e.g. `"--output"`, `"--json"`) |

When `"json"` is listed in `formats`, consumers that require structured output SHOULD pass the `formatFlag` with value `"json"` (or equivalent) to ensure parseable output.

---

## 13. Tags

Tags are free-form strings applied to the root document or individual commands for grouping and filtering. No tag vocabulary is mandated by this specification.

Conventional tags that consumers MAY give special meaning to:

| Tag | Suggested meaning |
|---|---|
| `"read"` | Command only reads; no writes or deletes |
| `"write"` | Command creates or modifies |
| `"dangerous"` | Extra caution warranted; often implies `intent.destructive` |
| `"admin"` | Requires elevated privileges |
| `"experimental"` | Unstable API; may change without notice |

---

## 14. Vendor Extensions

Any field prefixed with `x-` is a vendor extension. Extensions may appear on the root document and on the following objects: `command`, `namespace`, `parameter`, and `defaultHandler`. Extensions MUST NOT appear on other objects defined by this specification (`intent`, `output`, `deprecation`, `constraint`, `environment`, `envVar`, `configFile`) — these are closed vocabularies whose meaning is defined by this specification, and extensions there would dilute or fork that meaning. Tooling MAY attach extension data to the parent object instead (for example, attach LLM-hint metadata to the `command` rather than to its `intent`). Consumers MUST ignore unknown `x-` fields. Implementations MUST NOT use `x-` fields to override the semantics of standard fields.

```json
{
  "name": "deploy",
  "x-cloud-provider": "aws",
  "x-min-iam-role": "arn:aws:iam::123456789012:role/Deployer"
}
```

---

## 15. Security Considerations

Schema documents describe a program's interface; they do not execute code. Consumers should treat schema documents as untrusted input and validate them against the meta-schema before acting on their contents.

The `intent` object provides declarations made by the implementation author. Consumers SHOULD NOT assume that a command marked `destructive: false` is harmless — declarations may be incomplete or incorrect. Consumers SHOULD still apply their own risk reasoning.

Commands marked with `requiresAuth: true` may transmit credentials. Consumers that pass `confirmationSkip`-role flags on behalf of users MUST obtain explicit user authorization before doing so on destructive commands.

---

## Appendix A: Changelog

| Version | Changes |
|---|---|
| 1.0.0-draft | Initial draft |
